
+++
title = "Captured Variables in Closures and Async Blocks - Part Two"
date = 2026-05-23
description = "A mental model around ownership of captured variables when using closures and async blocks in Rust."
+++

This is Part Two of the "Captured Variables Closures and Async Blocks" post, so make sure to check out [Part One](@/blog/closures_and_async_blocks_1.md) if you haven't already, as Part Two builds on Part One.

## Recap from Part One
For reference, this is the implementation that we arrived at in Part One:
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

As mentioned in Part One, our first implementation still has a problem - it requires two redundant arguments for every write invocation, `object_writer` and `permits`. These instances will never change, so our API should ideally not require these to be specified for every call. We can solve this by creating a structure that remembers the `object_writer` and `permits` context. This way, every call would only been to include as arguments what's unique for that particular invocation, the `path` where the data will be stored and the `bytes` that represent the data.

As we saw in Part One, closures/async blocks can be thought of as structures with fields that correspond to the captured variables. With this in mind, we can modify our implementation to first create a closure that captures the `object_writer` and `permits` as its context, and then use this closure whenever we need to write an object. 

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

// Write multiple times (concurrently with backpressure) without needing to pass in neither the `object_writer` 
// nor the `permits`
writer_with_backpressure("/my_bucket/first_object", Bytes::from_static(b"1234")).await;
writer_with_backpressure("/my_bucket/second_object", Bytes::from_static(b"5678")).await;
```

Our new function takes in the context variables (`object_writer` and `permits`), and returns a function that takes for its input the unique arguments needed for each write invocation (`PathBuf` and `Bytes`). We kept all function argument types the same as Part One since we know those will work, at least as a good starting point. We are using an [`AsyncFn`](https://doc.rust-lang.org/stable/std/ops/trait.AsyncFn.html) instead of an [`Fn`](https://doc.rust-lang.org/stable/std/ops/trait.Fn.html) for the return type of this new function because we know that we'll eventually need to `await` a permit, but the rest of the function signature is straight-forward.

This approach is a lot more ergonomic than our first implementation. It makes code more readable (by needing fewer arguments), and more closely
aligns with our ideal design.

## Ideal Implementation
Next, let's implement our new function. For the most part, the contents of our new function will be the same as our first implementation, except that we'll wrap our implementation in a closure, which is the return value for our new function, like so:
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
We added `move` to the returned closure so that it takes ownership `object_writer` and `permits`, which is how we make sure that our closure has the context needed for future calls.

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
As its name suggests, [`AsyncFnOnce`](https://doc.rust-lang.org/stable/std/ops/trait.AsyncFnOnce.html) can only be called once because it
consumes itself when called. In other words, the compiler thinks `permits` is moved elsewhere as soon as the closure is called, making it unable to be called again.

## Building on our Mental Model
Let's use our mental model to understand this compiler error. We can think of our top-level closure (the return value of our new function) as the following:
```rust
struct TopLevelClosure {
    object_writer: impl ObjectWriter + Send + 'static,
    permits: Arc<Semaphore>,
}
```
The top-level closure owns its captured variables because we used `move`. This is by design since the top-level closure needs both `object_writer` and `permits` for it's context. Next let's take a look at the async block, which is the return value of our top-level closure and the next closure/async block in the hierarchy:
```rust
struct TopLevelAsyncBlock {
    permits: &Arc<Semaphore>
}
```
Using our mental model, the async block should capture `permits` by reference since we didn't use the `move` keyword. However, the following line in the compiler error gives us a hint that things might not be as we expect:
>  closure is `AsyncFnOnce` because it moves the variable `permits` out of its environment. 

In other words, `permits` is moved out of `TopLevelClosure`, even though we never explicitly use the `move` keyword to capture it elsewhere. However, if we examine how `permits` is used in the async block, we realize that [`permits.acquire_owned()`](https://docs.rs/tokio/latest/tokio/sync/struct.Semaphore.html#method.acquire_owned) takes an owned `Arc<Semaphore>`, which implicitly moves `permits` to satisfy that requirement. This explains the compiler error. If we could reword the compiler error using our specific code it would read something like:
> closure is `AsyncFnOnce` because `permits.acquire_owned()` in `TopLevelAsyncBlock` moves `permits` from `TopLevelClosure` into `TopLevelAyncBlock`.

{% note(type="tip") %}
The `move` keyword is not always necessary for captured variables to be moved into the closure structure. The compiler can decide to move them if the contents of the closure require an owned instance.
{% end %}

We can illustrate this problem using our mental model too. The top-level closure implementation would look like the following:
```rust
fn closure_fn(top_level_closure: TopLevelClosure) -> TopLevelAsyncBlock {
    TopLevelAyncBlock {
        permits: top_level_closure.permits,
    }
}
```
where an owned `TopLevelClosure` is required since `top_level_closure.permits` is being moved into `TopLevelAyncBlock`. This means `closure_fn` can only be called once because the `TopLevelClosure` gets dropped at the end of `closure_fn`.

We could solve this by cloning `permits` before calling `acquire_owned`, like so:
```rust
let permit = permits.clone().acquire_owned().await.unwrap();
```
however this approach would result in lifetime errors later. Async blocks should, in most cases, own the variables they capture. Because of the nature of asynchronous code, it's likely that the async block will execute after the captured variables have been dropped, which is not allowed in Rust.

Instead, we can clone `permits` before moving it into `TopLevelAsyncBlock` like so:
```rust
pub fn create_writer_with_backpressure(
    object_writer: impl ObjectWriter + Send + 'static,
    permits: Arc<Semaphore>,
) -> impl AsyncFn(PathBuf, Bytes) {
    move |path: PathBuf, bytes: Bytes| {
        let permits_clone = permits.clone();
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
which finally solves the `permits` compiler error. Using our mental model, we can now think of the top-level async block as the following:
```rust
struct TopLevelAsyncBlock {
    permits_clone: Arc<Semaphore>
}
```
and the top-level closure implementation as:
```rust
fn closure_fn(top_level_closure: &TopLevelClosure) -> TopLevelAsyncBlock {
    let permits_clone = top_level_closure.permits.clone();
    TopLevelAyncBlock {
        permits_clone,
    }
}
```
Note the use of a reference in the argument `&TopLevelClosure`. By using a reference, our mental-model `closure_fn` can be called multiple times, which is required for it to implement `AsyncFn`. Previously we couldn't use a reference because we needed to move `top_level_closure.permits` into `TopLevelAyncBlock`.

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
which is a similar error as the one we solved for `permits`. 
If we think of the captured variables by the spawned async block (which does use `move`) as:
```rust
struct SpawnedAsyncBlock {
    object_writer: impl ObjectWriter + Send + 'static,
    path: PathBuf, 
    bytes: Bytes,
    permit: OwnedSemaphorePermit,
}
```
we can reword the error for our code as:
> closure is `AsyncFnOnce` because it moves the variable `object_writer` from `TopLevelClosure` into `SpawnedAsyncBlock` due to the use of the `move` keyword.

We can fix this using the same strategy we used for `permits`. We'll clone `object_writer` before moving it into `SpawnedAsyncBlock`, like so:
```rust
pub fn create_writer_with_backpressure(
    object_writer: impl ObjectWriter + Clone + Send + 'static,
    permits: Arc<Semaphore>,
) -> impl AsyncFn(PathBuf, Bytes) {
    move |path: PathBuf, bytes: Bytes| {
        let permits_clone = permits.clone();
        let object_writer_clone = object_writer.clone();
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
Note that we also added a `Clone` bound to `object_writer`. Finally, no compiler errors!

## Recap
Let's analyze why our implementation works using our mental model. We can think of the top-level closure as:
```rust
fn closure_fn(top_level_closure: &TopLevelClosure, path: PathBuf, bytes: Bytes) -> TopLevelAsyncBlock {
    let permits_clone = top_level_closure.permits.clone();
    let object_writer_clone = top_level_closure.object_writer.clone();
    TopLevelAyncBlock {
        permits: permits_clone,
    }
}
```
and the top-level async block as:
```rust
fn top_level_async_block(
    top_level_closure: &TopLevelClosure, 
    top_level_async_block: TopLevelAsyncBlock,
    path: PathBuf, 
    bytes: Bytes,
) {
    let permit = top_level_async_block.permits.acquire_owned().await.unwrap();
    tokio::spawn(
        SpawnedAsyncBlock {
            
        }
    )
}
```
