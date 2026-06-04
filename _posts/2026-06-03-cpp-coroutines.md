---
layout: post
comments: false
title:  "Coroutines in C++"
excerpt: "A practical tour of C++20 coroutines, awaitables, promises, and coroutine frames"
date:   2026-06-03 00:00:00
---

C++20 coroutines are a language feature for pausing a function and resuming it later.
They are useful when a program has work that naturally waits: network requests,
disk I/O, timers, event loops, generators, and cooperative tasks. The important idea is that a coroutine is not a thread. A coroutine can suspend
without blocking the operating system thread that was running it. That makes it a
good fit for high-concurrency code where most tasks spend their time waiting for
external events.

### What makes a function a coroutine?

A function becomes a coroutine if its body contains one of these keywords:

- `co_await`: suspend until an awaited operation is ready.
- `co_yield`: produce a value and suspend.
- `co_return`: finish the coroutine and return through the coroutine machinery.

For example:

```cpp
task<int> fetch_count() {
    auto bytes = co_await read_socket();
    co_return parse_count(bytes);
}
```

This looks like ordinary sequential code, but the compiler transforms it into a
state machine. When `read_socket()` is not ready, `fetch_count()` can suspend.
Later, an event loop can resume it at the next line with the saved local state.

### The coroutine frame

When a coroutine is called, the compiler creates a coroutine frame. The frame
stores:

- parameters copied or moved into the coroutine,
- local variables that must survive suspension,
- the current suspension point,
- the promise object,
- bookkeeping needed to destroy or resume the coroutine.

This frame usually lives on the heap, although compilers may ignore the allocation
when the lifetime is clear. The function call returns to the caller with a
handle-like object, while the frame keeps the coroutine's state alive.

### Promise type

Every coroutine return type must provide a `promise_type`. The promise tells the
compiler how to build, suspend, return from, and clean up the coroutine.

A minimal task shape looks like this:

```cpp
#include <coroutine>
#include <exception>

struct task {
    struct promise_type {
        task get_return_object() {
            return task{
                std::coroutine_handle<promise_type>::from_promise(*this)
            };
        }
        std::suspend_never initial_suspend() noexcept { return {}; }
        std::suspend_always final_suspend() noexcept { return {}; }
        void return_void() noexcept {}
        void unhandled_exception() { std::terminate();}
    };

    std::coroutine_handle<promise_type> handle;

    explicit task(std::coroutine_handle<promise_type> h) : handle(h) {}

    task(const task&) = delete;
    task& operator=(const task&) = delete;

    task(task&& other) noexcept : handle(other.handle) {
        other.handle = {};
    }

    ~task() {
        if (handle) {
            handle.destroy();
        }
    }
};
```

There are several important choices here:

- `initial_suspend()` controls whether the coroutine starts immediately or waits
  for someone to resume it.
- `final_suspend()` controls what happens after the coroutine finishes.
- `return_void()` or `return_value()` receives the result of `co_return`.
- `unhandled_exception()` receives exceptions that escape the coroutine body.

This example is intentionally small. Real task types usually store results,
propagate exceptions, resume a continuation, and integrate with an executor or
event loop. 

### Awaitables

`co_await expr` works by turning `expr` into an awaiter. An awaiter is an object
that supports three operations:

```cpp
struct simple_awaiter {
    bool await_ready() const noexcept { return false; }
    void await_suspend(std::coroutine_handle<> h) {
        // Save h somewhere and resume it later.
    }
    int await_resume() noexcept { return 42; }
};
```

The flow is:

1. `await_ready()` is called first. If it returns `true`, the coroutine does not
   suspend.
2. `await_suspend(handle)` is called when suspension is needed. The awaiter gets
   the coroutine handle so it can arrange resumption.
3. `await_resume()` is called when the coroutine continues. Its return value
   becomes the result of the `co_await` expression.

This is the central extension point. A socket read, timer, mutex, task, or custom
scheduler can all become awaitable by implementing this protocol.

### A tiny generator

Coroutines are also useful for lazy sequences. A generator uses `co_yield` to
produce one value at a time:

```cpp
generator<int> range(int first, int last) {
    for (int value = first; value < last; ++value) {
        co_yield value;
    }
}
```

Conceptually, each `co_yield` stores a value in the promise and suspends. The
caller pulls values by resuming the coroutine until it is done.

Generators are often easier to understand than asynchronous tasks because there
is no event loop involved. The same coroutine mechanics are still underneath:
there is a frame, a promise, suspension points, and a handle used to resume
execution.

### Why not just use threads?

Threads and coroutines solve different problems.

Threads give parallel execution and are scheduled by the operating system.
Coroutines give structured suspension and are resumed by library code. A program
can have many more suspended coroutines than threads because a suspended
coroutine is mostly just a saved frame, not an OS scheduling resource.

Coroutines are especially useful when:

- the code waits on I/O more than it burns CPU,
- the program already has an event loop,
- callback chains are becoming hard to read,
- cancellation and lifetime need explicit structure,
- a lazy sequence should avoid materializing all values at once.

For CPU-bound parallel work, threads, thread pools, vectorization, and task
schedulers are usually the better starting point.

### Sharp edges

C++ coroutines are powerful but low-level. The standard gives the language
transformation, not a complete async runtime. That means the hardest parts are
usually in the library type:

- Who owns the coroutine handle?
- When is the frame destroyed?
- How are exceptions stored and rethrown?
- Which executor resumes the coroutine?
- What happens if an awaited operation is cancelled?
- Can the coroutine resume inline from `await_suspend()`?

These details matter because a coroutine handle is a raw capability to resume or
destroy suspended execution. Treat it like a resource-owning primitive, not like a
safe value type. The coroutines are also stackless and that means we can only suspend one function at a time. Then if we want to suspend a computation that spans other functions, we would have to suspend them all. 

### Mental model

The useful mental model is:

> A C++ coroutine is a compiler-generated state machine whose state is stored in
> a coroutine frame and controlled through a promise, awaiters, and a coroutine
> handle.

Once that model is clear, the keywords become less mysterious. `co_await` is a
structured suspension point. `co_yield` is a suspension point that publishes a
value. `co_return` completes the state machine. The return type and awaitable
types decide the runtime behavior around those points.



