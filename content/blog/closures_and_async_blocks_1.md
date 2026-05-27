+++
title = "Captured Variables in Closures and Async Blocks - Part One"
date = 2026-05-23
description = "A mental model around ownership of captured variables when using closures and async blocks in Rust."
+++

The goal of this post is to develop a mental model around ownership of captured variables when using closures and async blocks. This mental model will help you understand when and why you need to use [`move`](https://doc.rust-lang.org/stable/std/keyword.move.html), as well as how to interpret compiler errors involving closures and async blocks. This post is divided into two parts.

## Problem Statement
Let's dive deep into this topic by implementing a somewhat common pattern. Consider a service like AWS S3, which has a limitation on how many requests it can process simultaneously — let's call that limit `X`. To avoid errors on the `X + 1` request, we'll implement a writer with backpressure: the writer will concurrently execute up to `X` writes, but will wait before starting the `X + 1` write until at least one pending write completes.

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
We're leveraging [`Semaphore`s](https://docs.rs/tokio/latest/tokio/sync/struct.Semaphore.html), which are well-suited to our scenario. Before spawning the write task, we attempt to acquire a semaphore permit, which only yields once a permit is available. Then, in the spawned task, we explicitly drop the permit at the end to signal to the semaphore that a slot has freed up. We must reference `permit` inside the async block to ensure it is captured and moved in — without this, `permit` would be dropped in the outer scope immediately after the spawn, releasing the slot before the write completes. However, this approach has several issues, including compiler errors we'll address shortly.

## Implementation Issues
In addition to `path` and `bytes` (the two arguments unique to each invocation), the caller must also pass `object_writer` and `permits`. These instances never change across calls, so it's redundant to pass them on every invocation. We'll address this in [Part Two](@/blog/closures_and_async_blocks_2.md).

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
Let's break down the last line of the error:
> required for `&impl ObjectWriter` to implement `Send`

This requirement comes from [`tokio::spawn`](https://docs.rs/tokio/latest/tokio/task/fn.spawn.html). The future passed to `spawn` must satisfy:
```rust
Future + Send + 'static
```
How do we make a reference to `impl ObjectWriter` implement [`Send`](https://doc.rust-lang.org/stable/std/marker/trait.Send.html)? Notice that the compiler refers to a _reference_ to `impl ObjectWriter`, even though we started with an owned `impl ObjectWriter`. Why?

This is where we can start building a mental model around ownership and closures and async blocks.

## Building Our Mental Model
Closures and async blocks capture values by reference by default, unless the `move` keyword is used. What does this mean for our example?

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

Because we did _not_ use the `move` keyword, we can think of our async block as the following struct:
```rust
struct AsyncBlock<'a, 'b, 'c, 'd> {
    object_writer: &'a impl ObjectWriter,
    permits: &'b Arc<Semaphore>,
    path: &'c Path,
    bytes: &'d Bytes,
}
```
where all fields are references to the captured variables. This won't compile, but it's a useful mental model.

Then, the async block implementation can be thought of as the body of the async block wrapped in a function that takes an `AsyncBlock` instance:
```rust
async fn async_block_fn<...>(async_block: AsyncBlock<...>) {
    async_block
        .object_writer
        .write_object(async_block.path, async_block.bytes)
        .await;
    drop(async_block.permit);
}
```
In this view, every captured variable is a field of our `AsyncBlock` struct. When the async block is executed, the compiler passes a corresponding `AsyncBlock<...>` instance to `async_block_fn` to access those values. While this is not the actual compiled output, it gives us a clearer picture of how captured variables are used within the async block. Note that an async block [`Future`](https://doc.rust-lang.org/stable/std/future/trait.Future.html) can only be executed once, so in our mental model `AsyncBlock` is always passed by value.

This explains why the compiler sees a `&impl ObjectWriter` rather than an `impl ObjectWriter`: in our de-sugared view, `async_block.object_writer` is a field of type `&impl ObjectWriter`.

Back to the compiler error (skipping some detail):
```
error[E0277]: `impl ObjectWriter` cannot be shared between threads safely
    ...
    = note: required for `&impl ObjectWriter` to implement `Send`
```
How do we fix this? A reference to a type `T` is `Send` if `T` is `Sync`, so we could add a `Sync` bound to `object_writer`:
```rust
pub async fn write_with_backpressure(
    object_writer: impl ObjectWriter + Sync,
    ...
) {
    ...
}
```
While this resolves the compiler error, it doesn't actually work because it introduces another:
```
error[E0373]: async block may outlive the current function, but it borrows `object_writer`, which is owned by the current function
```

What we actually need is for the async block to take ownership of the captured values. We can do that with the `move` keyword:
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
This still does not compile, but we are getting closer. Now we get:
```
error: future cannot be sent between threads safely
   ...
note: captured value is not `Send`
    |
 29 |         object_writer.write_object(path, bytes).await;
    |         ^^^^^^^^^^^^^ has type `impl ObjectWriter` which is not `Send`
note: required by a bound in `tokio::spawn`
```
which can be resolved by adding a `Send` bound to `impl ObjectWriter`:
```rust
    ...
    object_writer: impl ObjectWriter + Send
    ...
```
More notably, the compiler now sees an `impl ObjectWriter` rather than an `&impl ObjectWriter`. This is a direct result of the `move` keyword.

Let's revisit our mental model with the `move` variant. When the `move` keyword is used on a closure or async block, the resulting struct owns the captured values rather than holding references to them. We can now think of our `move` async block as:
```rust
struct AsyncBlock<'a> {
    object_writer: impl ObjectWriter + Send,
    permits: Arc<Semaphore>,
    path: &'a Path,
    bytes: Bytes,
}
```
The field types exactly match the types of the captured variables. Note that `AsyncBlock` does not own a [`Path`](https://doc.rust-lang.org/stable/std/path/struct.Path.html) because the captured variable is of type `&Path` — the async block takes ownership of the reference, not of the `Path` itself.

Even after adding `Send`, the compiler is still not satisfied:
```
error[E0310]: the parameter type `impl ObjectWriter + Send` may not live long enough
   ...
help: consider adding an explicit lifetime bound
   |
29 |     object_writer: impl ObjectWriter + Send + 'static,
```
which can be resolved by adding a `'static` bound to `object_writer`.

{% note(type="tip") %}
Remember that `'static` used as a bound means that the type is an owned value (it doesn't have any references to other instances). It _doesn't_ mean that the variable will live for the duration of the program. I find this overloading of the `'static` keyword a bit unfortunate since it can be confusing.
{% end %}

Adding `Send` and `'static` as bounds to `impl ObjectWriter` is not unreasonable — most types conform to these, so we are not painting ourselves into a corner. Once we add them, the compiler error around `object_writer` goes away.

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
However, there are still compiler errors. Starting with:
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
This error mirrors the one we encountered with `object_writer`. Just like `impl ObjectWriter`, the `permit` variable must be `'static` — that is, it must be an owned value with no references to other instances. Looking at the type [`SemaphorePermit`](https://docs.rs/tokio/latest/tokio/sync/struct.SemaphorePermit.html), the problem becomes clear:
```rust
pub struct SemaphorePermit<'a> {
    sem: &'a Semaphore,
    permits: u32,
}
```
It contains a reference to a [`Semaphore`](https://docs.rs/tokio/latest/tokio/sync/struct.Semaphore.html), which makes it non-`'static` and therefore incompatible with [`tokio::spawn`](https://docs.rs/tokio/latest/tokio/task/fn.spawn.html). Fortunately, the Tokio authors anticipated this, and provided an owned variant. We can switch to [`acquire_owned`](https://docs.rs/tokio/latest/tokio/sync/struct.Semaphore.html#method.acquire_owned):
```rust
    ...
    let permit = permits.acquire_owned().await.unwrap();
    ...
```
which returns an [`OwnedSemaphorePermit`](https://docs.rs/tokio/latest/tokio/sync/struct.OwnedSemaphorePermit.html) that is `'static`, since it contains no references:
```rust
pub struct OwnedSemaphorePermit {
    sem: Arc<Semaphore>,
    permits: u32,
}
```

The next error follows the same pattern. [`Path`](https://doc.rust-lang.org/stable/std/path/struct.Path.html) is not `'static`, so we replace it with its owned counterpart, [`PathBuf`](https://doc.rust-lang.org/stable/std/path/struct.PathBuf.html).

With those changes, our implementation compiles without errors:
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

We still haven't addressed the redundancy noted earlier: `object_writer` and `permits` never change, yet must be passed on every call. We'll tackle that in [Part Two](@/blog/closures_and_async_blocks_2.md), which will give us another opportunity to explore closures and async blocks.
