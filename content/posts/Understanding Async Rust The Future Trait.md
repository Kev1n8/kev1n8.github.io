+++
title = "Understanding Async Rust: The Future Trait"
date = 2025-04-01

[taxonomies]
tags = ["rust"]
+++

# Understanding Async Rust: The Future Trait

This blog is intended to summarize the key ideas when it comes to async Rust, hence it's not going to deep in every topic. Please refer to more professional papers.

## General Picture of `async` and `await`

It's obvious that the biggest difference between sync and async code is that the latter one has these keywords used in function signatures or bodies. However, one may wonder what is that supposed to mean.

`async` is used in front of a `fn` or a closure. It literally means that the `fn` or the closure is returning a `Future`. Meanwhile, `await` is "called" on a `Future`. To simply say, it tells the compiler to generate necessary codes for state changing.

So the big picture is, we treat async functions (or async closures) as a mission to be completed for in the future. And it's not going to activate/be invoked unless we call `await` on that `Future`. While waiting for the result of that `Future`, the control is yielded to other tasks that might be ready and waiting to be executed. The "pause" and "resume" of tasks mechanism should thanks to a specific async task runtime. 

There are several great tools when it comes to async runtime for Rust, e.g. tokio, async-std, smol. Since tokio is the most used one, we'll focus on it.

A runtime is responsible for the reconsiling of tasks. In tokio, there are several workers, each runs on a thread, polling the futures (we'll explain "polling" later. you may think it's like processing that task). Each worker has a local queue whose items are tasks waiting to be polled. And there also is a global queue first. The task is stealing-based in tokio, which means if a worker has empty queue, it'll try to steal task from other workers' queue if available.

A simple example of usage:
```rust
async fn run_async_task() -> i32 {
    return 0;
}

#[tokio::main]
async fn main() {
    let fut = run_async_task(); // Return a future, not run.
    let zero = fut.await; // Here, the control yields and the runtime will start `run_async_task()`
}
```

## About `Future`

`Future` is a trait that indicates its result might be ready in the future. It's similar to promise in JavaScript. Let's look at its definition:

```rust
pub trait Future {
    /// The type of value produced on completion.
    type Output;
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
}
```

## Example

By the following example, you'll get how futrues work:

```rust
async fn fn_1() -> i32 {
    1i32
}
async fn add_2(a: i32) -> i32 {
    a + 2
}
async fn run_add() {
   let a = fn_1().await;
   let b = add_2(a).await;
   println!("{b}");
}
#[tokio::main]
async fn main() {
    run_add().await;
}
```

```rust
// Simplified pseudocode illustrating the state machine for run_add()
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll, Waker};

// Let's assume these async functions exist:
async fn fn_1() -> i32 {
    1 // Returns immediately for simplicity.
}

async fn add_2(a: i32) -> i32 {
    a + 2 // Also returns immediately for simplicity.
}

// The state machine states for run_add
enum RunAddState {
    Start,                   // Before any await.
    WaitingFn1 {              // Waiting for fn_1 to complete.
        future: Pin<Box<dyn Future<Output = i32>>>,
    },
    WaitingAdd2 { a: i32,      // Holds the result from fn_1.
        future: Pin<Box<dyn Future<Output = i32>>>,
    },
    Finished,
}

struct RunAdd {
    state: RunAddState,
}

impl Future for RunAdd {
    type Output = ();

    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<()> {
        loop {
            match &mut self.state {
                RunAddState::Start => {
                    // We initiate fn_1() here.
                    let future_fn1 = Box::pin(fn_1());
                    // Transition state to WaitingFn1 with the future.
                    self.state = RunAddState::WaitingFn1 { future: future_fn1 };
                }
                RunAddState::WaitingFn1 { future } => {
                    // Poll the inner future from fn_1.
                    match future.as_mut().poll(cx) {
                        Poll::Ready(a) => {
                            // fn_1() is done, we got the result 'a'.
                            // Now prepare to call add_2(a).
                            let future_add2 = Box::pin(add_2(a));
                            self.state = RunAddState::WaitingAdd2 { a, future: future_add2 };
                        }
                        Poll::Pending => {
                            // fn_1() isn't finished yet.
                            // Yield control back to the executor.
                            return Poll::Pending;
                        }
                    }
                }
                RunAddState::WaitingAdd2 { a: _, future } => {
                    // Poll the inner future from add_2.
                    match future.as_mut().poll(cx) {
                        Poll::Ready(b) => {
                            // add_2 has finished. We can now use the result.
                            println!("Result: {}", b);
                            self.state = RunAddState::Finished;
                            return Poll::Ready(());
                        }
                        Poll::Pending => {
                            // Still waiting for add_2—for now, yield control.
                            return Poll::Pending;
                        }
                    }
                }
                RunAddState::Finished => {
                    panic!("Polling after completion");
                }
            }
        }
    }
}

#[tokio::main]
async fn main() {
    RunAdd { state: RunAddState::Start }.await; // Runtime polls here.
}
```

Then, the async runtime will start "epolling" (kinda) the futures, until it returns `Finished`.

ref. https://wiki.cont.run/lowering-async-await-in-rust/
