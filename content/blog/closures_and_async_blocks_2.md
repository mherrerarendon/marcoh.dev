+++
title = "Captured Variables in Closures and Async Blocks - Part Two"
date = 2026-05-27
description = "A mental model around ownership of captured variables when using closures and async blocks in Rust."
+++

This is Part Two of the "Captured Variables in Closures and Async Blocks" series, so make sure to check out [Part One](@/blog/closures_and_async_blocks_1.md) if you haven't already, since Part Two builds directly on it.

## Recap from Part One
For reference, this is the implementation we arrived at in [Part One](@/blog/closures_and_async_blocks_1.md):
```rust
pub async fn write_with_backpressure(
    object_writer: impl ObjectWriter + Send + 'static,
    permits: Arc<Semaphore>,
    path: PathBuf,
    bytes: Bytes,
) {
    let permit = permits.acquire_owned().await.unwrap();
    tokio::spawn(async move {
        object_writer.write_object(path, bytes).await;
        drop(permit);
    });
}
```

As mentioned in [Part One](@/blog/closures_and_async_blocks_1.md), this implementation still has a problem — it requires two redundant arguments for every write invocation: `object_writer` and `permits`. These instances will never change, so our API should ideally not require them to be specified on every call. We can solve this by creating a structure that holds the `object_writer` and `permits` context. This way, every call would only need to include what is unique to that particular invocation: the `path` where the data will be stored and the `bytes` that represent the data.

As we saw in [Part One](@/blog/closures_and_async_blocks_1.md), closures and async blocks can be thought of as structs whose fields correspond to the captured variables. With this in mind, we can modify our implementation to first create a closure that captures `object_writer` and `permits` as its context, and then use this closure whenever we need to write an object.

## Ideal API
Let's start with our ideal function signature:
```rust
pub fn create_writer_with_backpressure(
    object_writer: impl ObjectWriter + Send + 'static,
    permits: Arc<Semaphore>,
) -> impl AsyncFn(PathBuf, Bytes) {
    ...
}
```
which can be used by a caller like so:
```rust
let writer_with_backpressure = create_writer_with_backpressure(object_writer, permits);

// Write multiple times (concurrently with backpressure) without needing to pass in either
// the `object_writer` or the `permits`
writer_with_backpressure("/my_bucket/first_object".into(), Bytes::from_static(b"1234")).await;
writer_with_backpressure("/my_bucket/second_object".into(), Bytes::from_static(b"5678")).await;
```

Our new function takes the context variables (`object_writer` and `permits`) and returns a function that takes as input the unique arguments needed for each write invocation ([`PathBuf`](https://doc.rust-lang.org/stable/std/path/struct.PathBuf.html) and [`Bytes`](https://docs.rs/bytes/latest/bytes/struct.Bytes.html)). We kept all argument types the same as in [Part One](@/blog/closures_and_async_blocks_1.md#remaining-implementation-issues) since we know those work, at least as a good starting point. We use [`AsyncFn`](https://doc.rust-lang.org/stable/std/ops/trait.AsyncFn.html) instead of [`Fn`](https://doc.rust-lang.org/stable/std/ops/trait.Fn.html) as the return type because we know we'll eventually need to `await` a permit, but the rest of the function signature is straightforward.

This approach is considerably more ergonomic than our first implementation. It makes code more readable by requiring fewer arguments, and more closely aligns with our ideal design.

## Ideal Implementation
Next, let's implement our new function. For the most part, the body of our new function will be the same as the first implementation from [Part One](@/blog/closures_and_async_blocks_1.md#remaining-implementation-issues), except that we'll wrap it in a closure that takes a [`PathBuf`](https://doc.rust-lang.org/stable/std/path/struct.PathBuf.html) and [`Bytes`](https://docs.rs/bytes/latest/bytes/struct.Bytes.html), which becomes the return value, like so:
```rust
pub fn create_writer_with_backpressure(
    object_writer: impl ObjectWriter + Send + 'static,
    permits: Arc<Semaphore>,
) -> impl AsyncFn(PathBuf, Bytes) {
    move |path: PathBuf, bytes: Bytes| async {
        let permit = permits.acquire_owned().await.unwrap();
        tokio::spawn(async move {
            object_writer.write_object(path, bytes).await;
            drop(permit);
        });
    }
}
```
We added `move` to the returned closure so that it takes ownership of `object_writer` and `permits`, ensuring our closure has the context it needs for future calls.

Let's see what the compiler thinks about our new implementation:
```
error[E0525]: expected a closure that implements the `AsyncFn` trait, but this closure only implements `AsyncFnOnce`
 9 |   ) -> impl AsyncFn(PathBuf, Bytes) {
   |        ---------------------------- the requirement to implement `AsyncFn` derives from here
10 |       move |path: PathBuf, bytes: Bytes| async {
   |       -^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
   |       |
   |  _____this closure implements `AsyncFnOnce`, not `AsyncFn`
   | |
11 | |         let permit = permits.acquire_owned().await.unwrap();
   | |                      ------- closure is `AsyncFnOnce` because it moves the variable `permits` out of its environment
12 | |         tokio::spawn(async move {
13 | |             object_writer.write_object(path, bytes).await;
14 | |             drop(permit);
15 | |         });
16 | |     }
   | |_____- return type was inferred to be `{closure@src/second.rs:10:5: 10:39}` here
```
Let's break down this specific line:
> closure is `AsyncFnOnce` because it moves the variable `permits` out of its environment

As its name suggests, [`AsyncFnOnce`](https://doc.rust-lang.org/stable/std/ops/trait.AsyncFnOnce.html) can only be called once because it consumes itself when called. In other words, the compiler determines that `permits` is moved away on the first call, making it unavailable for any subsequent calls.

## Building on our Mental Model
Let's use the mental model we started building in [Part One](@/blog/closures_and_async_blocks_1.md) to understand this compiler error. We can think of the captured variables for the returned closure as the following struct:
```rust
struct Closure {
    object_writer: impl ObjectWriter + Send + 'static,
    permits: Arc<Semaphore>,
}
```
The closure owns its captured variables because we used `move`. This is by design, since the closure needs both `object_writer` and `permits` for its context. Next, let's look at the async block returned by the closure:
```rust
struct AsyncBlock {
    permits: &Arc<Semaphore>
}
```
Using our mental model, we would expect the async block to capture `permits` by reference since we didn't use the `move` keyword. However, the following line from the compiler error hints that things may not be as we expect:
> closure is `AsyncFnOnce` because it moves the variable `permits` out of its environment.

In other words, `permits` is moved out of `Closure` even though we never explicitly used the `move` keyword. If we examine how `permits` is used inside the async block, we can see that [`permits.acquire_owned()`](https://docs.rs/tokio/latest/tokio/sync/struct.Semaphore.html#method.acquire_owned) takes an `Arc<Semaphore>` by value, which implicitly moves `permits` to satisfy that requirement. This explains the compiler error. Rephrased in terms of our mental model structs, the error would read:
> closure is `AsyncFnOnce` because `permits.acquire_owned()` in `AsyncBlock` moves `permits` from `Closure` into `AsyncBlock`.

{% note(type="tip") %}
The `move` keyword is not always necessary for a captured variable to be moved into the closure struct. The compiler will move it implicitly if the body of the closure requires an owned instance.
{% end %}

We can illustrate this problem using our mental model. The de-sugared closure implementation would look like the following:
```rust
fn closure_fn(closure: Closure) -> AsyncBlock {
    AsyncBlock {
        permits: closure.permits,
    }
}
```
As discussed in [Part One](@/blog/closures_and_async_blocks_1.md), this view gives us a clearer picture of how captured variables are used and moved within a closure. Here, the `Closure` argument must be passed by value because `closure.permits` is moved into `AsyncBlock`. This means `closure_fn` can only be called once, since `closure` is consumed at the end of the call.

We could attempt to solve this by cloning `permits` before calling `acquire_owned`, like so:
```rust
let permit = permits.clone().acquire_owned().await.unwrap();
```
however, this approach would result in lifetime errors later.

{% note(type="tip") %}
Async blocks should, in most cases, own the variables they capture. Because of the nature of asynchronous code, it is likely that an async block will outlive the scope in which the captured variables were defined, which Rust does not permit.
{% end %}

Instead, we can clone `permits` _before_ moving it into `AsyncBlock`, like so:
```rust
pub fn create_writer_with_backpressure(
    object_writer: impl ObjectWriter + Send + 'static,
    permits: Arc<Semaphore>,
) -> impl AsyncFn(PathBuf, Bytes) {
    move |path: PathBuf, bytes: Bytes| {
        let permits_clone = permits.clone(); // this line is new
        async move {
            let permit = permits_clone.acquire_owned().await.unwrap();
            tokio::spawn(async move {
                object_writer.write_object(path, bytes).await;
                drop(permit);
            });
        }
    }
}
```
which finally resolves the `permits` compiler error. Using our mental model, we can now think of the async block as:
```rust
struct AsyncBlock {
    permits_clone: Arc<Semaphore>
}
```
and the closure implementation as:
```rust
fn closure_fn(closure: &Closure) -> AsyncBlock {
    let permits_clone = closure.permits.clone();
    AsyncBlock {
        permits_clone,
    }
}
```
Note the use of a reference in the argument `&Closure`. By taking a reference, `closure_fn` can be called multiple times, which is required for it to implement `AsyncFn`. Previously we couldn't use a reference because we needed to move `closure.permits` into `AsyncBlock`.

## Remaining Implementation Issues
Our next compiler error is the following:
```
error[E0525]: expected a closure that implements the `AsyncFn` trait, but this closure only implements `AsyncFnOnce`
 9 |   ) -> impl AsyncFn(PathBuf, Bytes) {
   |        ---------------------------- the requirement to implement `AsyncFn` derives from here
10 |       move |path: PathBuf, bytes: Bytes| {
   |       -^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
   |       |
   |  _____this closure implements `AsyncFnOnce`, not `AsyncFn`
   | |
11 | |         let permits_clone = permits.clone();
12 | |         async move {
13 | |             let permit = permits_clone.acquire_owned().await.unwrap();
14 | |             tokio::spawn(async move {
15 | |                 object_writer.write_object(path, bytes).await;
   | |                 ------------- closure is `AsyncFnOnce` because it moves the variable `object_writer` out of its environment
...  |
19 | |     }
   | |_____- return type was inferred to be `{closure@src/second.rs:10:5: 10:39}` here
```
where the key line is:
> closure is `AsyncFnOnce` because it moves the variable `object_writer` out of its environment

This is similar to the error we solved for `permits`. If we think of the captured variables of the spawned async block as:
```rust
struct SpawnedAsyncBlock {
    object_writer: impl ObjectWriter + Send + 'static,
    path: PathBuf,
    bytes: Bytes,
    permit: OwnedSemaphorePermit,
}
```
we can reword the error in terms of our mental model structs as:
> closure is `AsyncFnOnce` because it moves `object_writer` from `Closure` into `SpawnedAsyncBlock` due to the `move` keyword.

We can fix this using the same strategy we used for `permits` — clone `object_writer` before moving it into `SpawnedAsyncBlock`, like so:
```rust
pub fn create_writer_with_backpressure(
    object_writer: impl ObjectWriter + Clone + Send + 'static,
    permits: Arc<Semaphore>,
) -> impl AsyncFn(PathBuf, Bytes) {
    move |path: PathBuf, bytes: Bytes| {
        let permits_clone = permits.clone();
        let object_writer_clone = object_writer.clone(); // this line is new
        async move {
            let permit = permits_clone.acquire_owned().await.unwrap();
            tokio::spawn(async move {
                object_writer_clone.write_object(path, bytes).await;
                drop(permit);
            });
        }
    }
}
```
Note that we also added a `Clone` bound to `object_writer`. With that, the compiler is satisfied.

## Recap
Let's analyze why our final implementation works by de-sugaring the closure and async blocks using our mental model. We can think of the closure implementation as:
```rust
fn closure_fn(closure: &Closure) -> AsyncBlock {
    let permits_clone = closure.permits.clone();
    let object_writer_clone = closure.object_writer.clone();
    AsyncBlock {
        permits_clone,
        object_writer_clone,
    }
}
```
The closure generates a new `AsyncBlock` instance every time it's called, which is why it needs to clone `permits` and `object_writer` and move them into `AsyncBlock`. Because `closure_fn` only takes a reference to `Closure`, it can be called multiple times — a requirement for our API.

Then, the `AsyncBlock` instance generated by the closure call gets passed by value into the following de-sugared async block implementation:
```rust
fn async_block_fn(async_block: AsyncBlock) {
    let permit = async_block.permits_clone.acquire_owned().await.unwrap();
    tokio::spawn(
        SpawnedAsyncBlock {
            object_writer_clone: async_block.object_writer_clone,
        }
    )
}
```
As we saw in [Part One](@/blog/closures_and_async_blocks_1.md), async block [`Future`s](https://doc.rust-lang.org/stable/std/future/trait.Future.html) can only be used once (unlike a closure, which can be called multiple times), which is why `AsyncBlock` is passed by value. This also allows us to move `async_block.object_writer_clone` into `SpawnedAsyncBlock` without needing to clone it again.

Lastly, the `SpawnedAsyncBlock` instance gets passed by value into the following de-sugared implementation:
```rust
fn spawned_async_block_fn(spawned_async_block: SpawnedAsyncBlock) {
    spawned_async_block
        .object_writer_clone
        .write_object(spawned_async_block.path, spawned_async_block.bytes)
        .await;
    drop(spawned_async_block.permit);
}
```

Thank you for reading! I hope this two-part series has given you a clearer mental model for reasoning about ownership in Rust closures and async blocks.
