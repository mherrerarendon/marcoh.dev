
+++
title = "Closures and Async Blocks - Part Two"
date = 2026-05-23
description = "A mental model around ownership when using closures and async blocks in Rust."
+++

This is Part Two of the "Closures and Async Blocks" post, so make sure to check out [Part One](@/blog/closures_and_async_blocks_1.md) if you haven't already, as Part Two builds on Part One.

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
We added `move` to the returned closure so that it takes ownership `object_writer` and `permits` (these are the captured varialbes), which is how we make sure that our closure has the context needed for future calls.

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

Let's use our mental model to understand this compiler error. We can think of our top-level closure (the return value of our new function) as the following:
```rust
struct TopLevelClosure {
    object_writer: impl ObjectWriter + Send + 'static,
    permits: Arc<Semaphore>,
}
```
The top-level closure owns its captured variables because we used `move`. So far so good. Now let's take a look at the async block, which is the return value of our top-level closure:
```rust
struct AsyncBlockReturnedByTopLevelClosure {
    permits: &Arc<Semaphore>
}
```
The problem isn't readily apparent yet. However, if we examine how `AsyncBlockReturnedByTopLevelClosure::permits` is used in the async block contents, we realize that the [`permits.acquire_owned()`](https://docs.rs/tokio/latest/tokio/sync/struct.Semaphore.html#method.acquire_owned) call requires an owned `Arc<Semaphore>`, but we only have a reference. Because of this discrepancy, the compiler will attempt to `move` `permits` into `AsyncBlockReturnedByTopLevelClosure`, resulting in the following structure:
```rust
struct AsyncBlockReturnedByTopLevelClosure {
    permits: Arc<Semaphore>
}
```
which explains the compiler error. `permits` is being moved from `TopLevelClosure` to `AsyncBlockReturnedByTopLevelClosure`, because the contents of the async block required an owned `Arc<Semaphore>`. This makes our closure unable to run more than once, which is why it is only an `AsyncFnOnce`.

{% note(type="tip") %}
The `move` keyword is not always necessary for captured variables to be moved into the closure structure. The compiler can decide to move them even if the `move` keyword is absent, depending on how the captured variable is used.
{% end %}
