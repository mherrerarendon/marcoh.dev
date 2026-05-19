+++
title = "Building a High-Performance Async Stack"
date = 2026-05-18
description = "A deep dive into building lightweight systems software."
+++

Typography is the foundation of the minimalist web. When you strip away trackers, cookie banners, heavy JS frameworks, and layout shifts, what remains is the pure text.

### Code Block Testing

Here is an example of an asynchronous execution loop using standard markdown code highlighting syntax:

```rust
#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    println!("Hello from a fast, minimal Zola site!");
    Ok(())
}
