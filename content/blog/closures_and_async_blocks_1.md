+++
title = "Captured Variables in Closures and Async Blocks - Part One"
date = 2026-05-23
description = "A mental model around ownership of captured variables when using closures and async blocks in Rust."
+++

The purpose of this post is to come away with a mental model around ownership of captured variables when using closures and async blocks. This mental model will help you understand why you need to use [`move`](https://doc.rust-lang.org/stable/std/keyword.move.html) in some circumstances, as well as gain a better understanding of compiler errors regarding closures and async blocks. This post is divided into two parts.

## Problem Statement
Let's deep dive into this topic by implementing a somewhat common pattern. Consider a service like AWS S3 - S3 has a limitation that it can only process `X` simultaneous requests. In order to avoid S3 erroring out on the `X + 1` request, we'll implement a writer with backpressure, where the writer will concurrently execute up to `X` writes, but will wait to do the `X + 1` write until at least one pending write completes.

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
We're leveraging [`Semaphore`s](https://docs.rs/tokio/latest/tokio/sync/struct.Semaphore.html) which are perfect for our scenario. Before spawning the write task, we attempt to acquire a semaphore permit, which will only yield once there is a permit available. Then, in the spawned task, the last thing it does is explicitly drop the permit, which signals to the semaphore that a new permit is available (we need to explicitly drop so that the `permit` variable is captured in the async block). This is how we achieve a maximum number of concurrent writes. However, there are a number of issues with this approach (including compiler errors, which we'll address a bit later).

## Implementation Issues
In addition to passing in the `path` and `bytes` (which are the two basic requirements for each unique invocation of the function), the caller has to also pass in the `object_writer` and the `permits`. The `object_writer` and `permits` instances will never change, so it's redundant to pass these for every write invocation. We'll try to fix this in our next iteration in [Part Two](@/blog/closures_and_async_blocks_2.md).

This code also fails to compile with the following errors:
```
error[E0277]: `impl ObjectWriter` cannot be shared between threads safely
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

This requirement comes from [`tokio::spawn`](https://docs.rs/tokio/latest/tokio/task/fn.spawn.html). The future passed in to `spawn` needs to be:
```rust
Future + Send + 'static
```
How do we make a reference to `impl ObjectWriter` type [`Send`](https://doc.rust-lang.org/stable/std/marker/trait.Send.html)? Notice that the compiler wants to use a _reference_ to `impl ObjectWriter`, even though we started with an owned `impl ObjectWriter`. Why?

This is where we can start building a mental model around ownership and closures/async blocks.

## Building Our Mental Model
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
struct AsyncBlock<'a, 'b, 'c, 'd> {
    object_writer: &'a impl ObjectWriter,
    permits: &'b Arc<Semaphore>,
    path: &'c Path,
    bytes: &'d Bytes,
}
```
where all fields are references to the captured variables. This won't compile, but it's helpful to think of it this way for our mental model.

Then, the async block implementation can be thought of as the following:
```rust
async fn async_block_fn<...>(async_block: AsyncBlock<...>) {
    async_block
        .object_writer
        .write_object(async_block.path, async_block.bytes)
        .await;
    drop(async_block.permit);
}
```
where every captured variable is actually just a field of our `AsyncBlock` struct.

Now we understand why the compiler thinks it has a `&impl ObjectWriter` instead of a `impl ObjectWriter`. If we use our mental model version of the async block contents, we can see that the compiler has access to `async_block.object_writer` which is a field of type `&impl ObjectWriter`.

Back to the compiler error (skipping some of the error detail):
```
error[E0277]: `impl ObjectWriter` cannot be shared between threads safely
    ...
    = note: required for `&impl ObjectWriter` to implement `Send`
```
How do we actually fix this error? A reference to a type `T` is `Send` if `T` is `Sync`, so we could solve our compiler error by making our `object_writer` implement `Sync` like so:
```rust
pub async fn write_with_backpressure(
    object_writer: impl ObjectWriter + Sync,
    ...
) {
    ...
}
```
While this solves the compiler error, it doesn't really work because it introduces another error:
```
error[E0373]: async block may outlive the current function, but it borrows `object_writer`, which is owned by the current function
```

What we actually need to do is for the async block to take ownership of the captured values, which we can do by using the `move` keyword, like so:
```rust
pub async fn write_with_backpressure(
    ...
) {
    let permit = permits.acquire().await.unwrap();
    tokio::spawn(async move {
        ...
    });
}
```
This still does not compile, but we are getting closer. Now we get the following error:
```
error: future cannot be sent between threads safely
   ...
note: captured value is not `Send`
    |
 29 |         object_writer.write_object(path, bytes).await;
    |         ^^^^^^^^^^^^^ has type `impl ObjectWriter` which is not `Send`
note: required by a bound in `tokio::spawn`
```
which can be solved by adding a `Send` bound to our `impl ObjectWriter` type.
```rust
    ...
    object_writer: impl ObjectWriter + Send
    ...
```
But more notably, the compiler now has an `impl ObjectWriter` instead of an `&impl ObjectWriter` like it did before. This is due to the `move` keyword.

Let's come back to our mental model but now with the `move` variant. When we use the `move` keyword on a closure/async block, the structure that represents the closure/async block will, from that point on, own the instances that it captures. We can now think of our `move` async block as the following structure:
```rust
struct AsyncBlock<'a> {
    object_writer: impl ObjectWriter + Send,
    permits: Arc<Semaphore>,
    path: &'a Path,
    bytes: Bytes,
}
```
The field types in our new structure exactly match the type of the variables that were captured. Note that our `AsyncBlock` struct does not own a [`Path`](https://doc.rust-lang.org/stable/std/path/struct.Path.html) because the captured variable is of type `&Path`. So the async block takes ownership of that reference, not of the `Path` instance.

However, maybe not unsurprisingly, the compiler is still not happy. Even after making our `object_writer` `Send`, we get this compiler error:
```
error[E0310]: the parameter type `impl ObjectWriter + Send` may not live long enough
   ...
help: consider adding an explicit lifetime bound
   |
29 |     object_writer: impl ObjectWriter + Send + 'static,
```
which again can be easily solved by making our `object_writer` have a `'static` bound.

{% note(type="tip") %}
Remember that `'static` used as a bound means that the type is an owned value (it doesn't have any references to other instances). It _doesn't_ mean that the variable will live for the duration of the program. I find this overloading of the `'static` keyword a bit unfortunate since it can be confusing.
{% end %}

Adding `Send` and `'static` as bounds to our `impl ObjectWriter` type is not unreasonable. Most types conform to these bounds, so we are not painting ourselves into a corner by adding these. Once we add these two bounds, the compiler error around `object_writer` goes away.

## Remaining Implementation Issues
Our implementation now looks like the following:
```rust
pub async fn write_with_backpressure(
    object_writer: impl ObjectWriter + Send + 'static,
    permits: Arc<Semaphore>,
    path: &Path,
    bytes: Bytes,
) {
    let permit = permits.acquire().await.unwrap();
    tokio::spawn(async move {
        object_writer.write_object(path, bytes).await;
        drop(permit);
    });
}
```
However, there are still some compiler errors. Starting with:
```
error[E0597]: `permits` does not live long enough
 30 |       permits: Arc<Semaphore>,
    |       ------- binding `permits` declared here
...
 34 |       let permit = permits.acquire().await.unwrap();
    |                    ^^^^^^^ borrowed value does not live long enough
 35 | /     tokio::spawn(async move {
 36 | |         object_writer.write_object(path, bytes).await;
 37 | |         drop(permit);
 38 | |     });
    | |______- argument requires that `permits` is borrowed for `'static`
 39 |   }
    |   - `permits` dropped here while still borrowed
    |
note: requirement that the value outlives `'static` introduced here
   --> /tokio-1.52.3/src/task/spawn.rs:176:28
    |
176 |         F: Future + Send + 'static,
    |                            ^^^^^^^
```
This error might look familiar. We encountered a similar version of this error when fixing the errors corresponding to `object_writer`. Just like with the `impl ObjectWriter` type, the `permits` variable needs to be `'static`. In other words, it needs to be an owned value and cannot contain references to other instances. However, if we take a look at the `permit` type, [`SemaphorePermit`](https://docs.rs/tokio/latest/tokio/sync/struct.SemaphorePermit.html), we can see the problem:
```rust
pub struct SemaphorePermit<'a> {
    sem: &'a Semaphore,
    permits: u32,
}
```
it contains a reference to a [`Semaphore`](https://docs.rs/tokio/latest/tokio/sync/struct.Semaphore.html). This means that this type is not `'static` and therefore incompatible with [`tokio::spawn`](https://docs.rs/tokio/latest/tokio/task/fn.spawn.html), which requires the passed in `Future` to be `'static`. Fortunately, the authors of the [`tokio`](https://docs.rs/tokio/latest/tokio/index.html) library are aware of this limitation, and wrote an owned variant of a semaphore permit. We can change our code to use [`acquire_owned`](https://docs.rs/tokio/latest/tokio/sync/struct.Semaphore.html#method.acquire_owned) instead:
```rust
    ...
    let permit = permits.acquire_owned().await.unwrap();
    ...
```
which is `'static` since it does not contain any references:
```rust
pub struct OwnedSemaphorePermit {
    sem: Arc<Semaphore>,
    permits: u32,
}
```

The next error is similar to the one we just solved. [`Path`](https://doc.rust-lang.org/stable/std/path/struct.Path.html) is not `'static`, so we need to use the owned version of `Path`, which is [`PathBuf`](https://doc.rust-lang.org/stable/std/path/struct.PathBuf.html).

Finally, our implementation does not have any compiler errors:
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

We still haven't solved the issue that we mentioned near the beginning of the post, which is that passing the `object_writer` and `permits` variables to each write invocation is redundant, since these instances will always be the same. We'll solve that in [Part Two](@/blog/closures_and_async_blocks_2.md) of this blog post, which conveniently will require more closures and async blocks.
