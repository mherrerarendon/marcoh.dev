+++
title = "Closures and Async Blocks"
date = 2026-05-19
description = "A mental model around ownership when using closures and async blocks."
+++

Why do I sometimes need to add `move` to a closure or async block?

The purpose of this post is to come away with a mental model around ownership when using closures and async blocks. This mental model will help you understand why you need to use `move` in some circumstances, as well as a better understand compiler errors regarding closures and async blocks.

## Problem Statement

Let's deep dive into this topic by implementing a some-what common pattern. Consider a service like AWS S3 - S3 has a limitation that it can only process X simultaneous requests. In order to avoid S3 erroring out on the X + 1 request, let's implement a writer with backpressure, where the writer will concurrently execute X writes, but will wait to do the X + 1 write until at least one previous write completes. Our first implementation could look something like the following:
```rust
```
In order to deep dive into the post's topic, we'll implement a writer with backpressure. Our writer with backpressure will allow us to write multiple things concurrently, but without exceeding a maximum number of concurrent writes. You can imagine this being helpful for an object storage service - you might want multiple requests to be able to write  Consider the following example

```rust

```

### Code Block Testing

Here is an example of an asynchronous execution loop using standard markdown code highlighting syntax:

```rust
#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    println!("Hello from a fast, minimal Zola site!");
    Ok(())
}
