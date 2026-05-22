+++
title = "Closures and Async Blocks"
date = 2026-05-19
description = "A mental model around ownership when using closures and async blocks."
+++

The purpose of this post is to come away with a mental model around ownership when using closures and async blocks. This mental model will help you understand why you need to use `move` in some circumstances, as well as a better understand compiler errors regarding closures and async blocks.

## Problem Statement

Let's deep dive into this topic by implementing a somewhat common pattern. Consider a service like AWS S3 - S3 has a limitation that it can only process X simultaneous requests. In order to avoid S3 erroring out on the X + 1 request, we'll implement a writer with backpressure, where the writer will concurrently execute X writes, but will wait to do the X + 1 write until at least one pending write completes. 

Assume that we have an S3 object that implements the following trait:
```rust
#[async_trait]
trait ObjectWriter {
    async fn write_object(&self, path: impl AsRef<Path>, bytes: Bytes);
}
```

## First Implementation
Our first implementation could look something like the following:
```rust
async fn write_with_backpressure(
    object_writer: impl ObjectWriter,
    permits: Arc<Semaphore>,
    path: &Path,
    bytes: Bytes,
) {
    let permit = permits.acquire().await.unwrap();
    tokio::spawn(async {
        object_writer.write_object(path, bytes).await;
        drop(permit);
    });
}
```
We're leveraging [`Semaphore`s](https://docs.rs/tokio/latest/tokio/sync/struct.Semaphore.html) which are perfect for our scenario. Before spawning the write task, we attempt to acquire a semaphore permit, which will only yield once there is a permit available. Then, in the spawned task, the last thing it does is drop the permit, which indicates the semaphore that a new permit is available. This is how we achieve a maximum number of concurrent writes. However, there are a number of issues with this approach (including compiler errors, which we'll address a bit later).

### Implementation Issues
In addition to passing in the `path` and `bytes` (which are the 2 basic requirements for each unique invocation of the function), the caller has to also pass in the `object_writer` and the `permits`. The `object_writer` and `permits` instances will never change, so it's redundant to pass these for every write invocation. We'll try to fix this in our next iteration.

This code also fail to compile with the following errors:
```
error[E0277]: `impl ObjectWriter` cannot be shared between threads safely
   --> src/first.rs:13:5
    |
 13 | /     tokio::spawn(async {
 14 | |         object_writer.write_object(path, bytes).await;
 15 | |         drop(permit);
 16 | |     });
    | |______^ `impl ObjectWriter` cannot be shared between threads safely
    |
    = note: required for `&impl ObjectWriter` to implement `Send`
```
Let's break down the last line of the error
> required for `&impl ObjectWriter` to implement `Send`

This requirement comes from [`tokio::spawn`](https://docs.rs/tokio/latest/tokio/task/fn.spawn.html). The future passed in to `spawn` needs to be 
```rust
Future + Send + 'static
```
How do we make a reference to `impl ObjectWriter` type `Send`? Notice that the compiler wants to use a _reference_ to `impl ObjectWriter`, even though we started with an owned `impl ObjectWriter`. Why?

This is where we can start building a mental model around ownership and closures/async blocks.

### Building Our Mental Model
Closures/async blocks will capture values by reference by default, unless the `move` keyword is used. What does this mean for our example? 

For reference, here's our async block: 
```rust
async {
    object_writer.write_object(path, bytes).await;
    drop(permit);
}
```
Our async block captures the following variables:
* `object_writer`
* `path`
* `bytes`
* `permit`

Because we did _not_ use the `move` keyword, we can think of our async block as the following structure:
```rust
struct AsyncBlock {
    object_writer: &impl ObjectWriter,
    permits: &Arc<Semaphore>,
    path: &Path,
    bytes: &Bytes,
}
```
where all fields are references to the captured variables. This won't compile, but it's helpful to think of it this way for our mental model.

Then, the async block implentation can be thought of as the following:
```rust
async fn async_block_fn(async_block: AsyncBlock) {
    async_block
        .object_writer
        .write_object(async_block.path, async_block.bytes)
        .await;
    drop(async_block.permit);
}
```
where every captured variable is actually just a field of our `AsyncBlock` struct.

We may start to realize now why the compiler thinks it has a `&impl ObjectWriter` instead of a `impl ObjectWriter`
