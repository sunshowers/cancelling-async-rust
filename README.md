# Cancelling async Rust

This repo contains notes and links related to my RustConf 2025 talk, _Cancelling async Rust_.

* [Slides](https://docs.google.com/presentation/d/e/2PACX-1vTMc4EdHRf6ulz-xaAhZFGZwxJ7jPQgYWczT6pEIvwfXILV4ZEgMdLuoRh70bgh9SP7mxblEnyXuZD0/pub?start=false&loop=false&delayms=60000)
* Video (coming soon!)
* Follow me on [Bluesky](https://bsky.app/profile/sunshowers.io) and [Mastodon](https://hachyderm.io/@rain)

Links to my blog:

* [My RustConf talk on Unix signals](https://sunshowers.io/posts/beyond-ctrl-c-signals/)
* [How (and why) nextest uses Tokio](https://sunshowers.io/posts/nextest-and-tokio/)

## Notes

For more in-depth discussion of issues related to cancellation, see these two Oxide RFDs:

* [RFD 397 Challenges with async/await in the control plane](https://rfd.shared.oxide.computer/rfd/397)
* [RFD 400 Dealing with cancel safety in async Rust](https://rfd.shared.oxide.computer/rfd/400)

### Why are futures passive?

The inert or passive nature of futures is a consequence of Rust async targeting embedded environments and other scenarios without memory allocation.

Think about it: If you want to run futures actively, the runtime must drive those futures -- for that, you must be able to move them off the stack and into the background. In full-fledged computing environments, that can be done by moving the future to the heap through e.g. [`Box::pin`](https://doc.rust-lang.org/std/boxed/struct.Box.html#method.pin). Tokio does this when you call [`tokio::task::spawn`](https://docs.rs/tokio/latest/tokio/task/fn.spawn.html). But in embedded no-alloc environments, the heap isn't available. The only options are:

1. Storing futures on the stack.
2. Moving them to static storage which must be declared at compile time.

Option 2 is of very limited use, so option 1 is the only viable path for arbitrary futures. Thus, futures are passive.

### Cooperative cancellation: cancellation tokens

Rather than cancellation at _any_ await point, what if you could allow cancellation at _specific_ await points? This can be done via a `select` operation using a _cancellation token_.

The tokio-util crate has an implementation of [`CancellationToken`](https://docs.rs/tokio-util/latest/tokio_util/sync/struct.CancellationToken.html). But note that by itself, this doesn't prevent cancellation at other await points. You must use other means (e.g. spinning up tasks) to avoid cancellation at other spots.

I've written [an implementation](https://docs.rs/cancel-safe-futures/latest/cancel_safe_futures/coop_cancel/index.html) of cancellation tokens as well. My implementation allows arbitrary data to be sent along with the cancellation message, but only permits one receiver (it uses an MPSC channel underneath).

### Actor model as an alternative to Tokio mutexes

As outlined in the talk, Tokio mutexes are very difficult to use correctly. The recommended alternative is to use a message-passing design, where each bit of mutable state has a single owner running within an independent task, and the owner communicates with the rest of the system through channels. This is often called the _actor model_.

The advantage of the actor model is that it is highly resilient to cancellation issues. The only way to cancel in-progress work is to abort the task, which might make some state invalid but will also likely tear down the task, tearing down this invalid state. (The exception here is state that's external to the task, such as external processes or services.)

For more discussion, see the [Tokio tutorial on channels](https://tokio.rs/tokio/tutorial/channels).

### Task aborts

I mentioned in the talk that Tokio's tasks are active and don't get cancelled on drop. However, it _is_ possible to cancel a Tokio task at the next await point using an [abort handle](https://docs.rs/tokio/latest/tokio/task/struct.AbortHandle.html). If your goal in creating a task is to run some cancel-unsafe code, be sure to not create any abort handles pointing to the task.

Tasks are also aborted (and therefore cancelled) on runtime shutdown (if using `#[tokio::main]`, this corresponds with process shutdown). If you don't want that to happen, you'll need to take the Tokio runtime's lifecycle into your own hands. (For example, find a way to block shutdown until all tasks are completed).

### Structured concurrency

A term often bandied about in these conversations is [_structured concurrency_](https://en.wikipedia.org/wiki/Structured_concurrency). Put succinctly, structured concurrency means that if a parent future creates a child future, the parent future waits for children to complete before it completes.

With structured concurrency, cancellation can be handled in two ways:

* **If the parent future is cancelled, all child futures are immediately cancelled as well.** This is how Rust futures work today (and what the talk is about!)
* **Cancellation of the parent future waits until child futures have exited.** This is not supported by Rust today.

Another way to do structured concurrency is to use a [nursery pattern](https://vorpus.org/blog/notes-on-structured-concurrency-or-go-statement-considered-harmful/#nurseries-a-structured-replacement-for-go-statements) as seen in [Trio](https://github.com/python-trio/trio). In Rust terms, that would be akin to the following strategy:

* Create a Tokio runtime.
* Start tasks on the runtime.
* On exit, wait until all tasks are completed.

This has some beneficial properties with respect to cancellation. See [_Timeouts and cancellation for humans_](https://vorpus.org/blog/timeouts-and-cancellation-for-humans) by Trio's author, [Nathaniel J. Smith](https://vorpus.org/), for more discussion.

### Relationship to panic safety

As mentioned in the talk, one way to cancel synchronous Rust code is to panic. But panicking has many of the same issues with invariant violations that async cancellations do.

To lower the likelihood of bugs, Rust's [`std::sync::Mutex`](https://doc.rust-lang.org/std/sync/struct.Mutex.html) implements a _poisoning_ scheme. If a panic occurs while the mutex is held, the next time a thread acquires the mutex it will be in the poisoned state. It is possible to [clear a mutex's poisoned state](https://doc.rust-lang.org/std/sync/struct.Mutex.html#method.clear_poison) in case you've restored invariants.

Tokio's mutexes do not implement poisoning, either for panics or for async cancellations, which makes them doubly dangerous.

### Async drop

One leading proposal to systematically address async cancellation issues in Rust is _async drop_. Currently, `Drop` impls must only contain synchronous code. With async drop, the `Drop` impl can also contain async code.

The ability to run async code on drop would address some of the issues brought up in the talk (e.g. running cleanup code on drop), but it wouldn't address issues where dropping the future is an invalid operation and should not be allowed (e.g. Tokio mutex-caused temporarily invalid state).

For more on async drop, see [this documentation](https://rust-lang.github.io/async-fundamentals-initiative/roadmap/async_drop.html).

### Unforgettable and undroppable types

It is often said that Rust has linear types, but in reality that is not true: Rust has what is known as _affine types_. The difference is that affine types can be used _at most once_, while linear types must be used _exactly once_[^must-use].

[^must-use]: What about `#[must_use]`? That is a _lint_ which makes it less likely that you'll drop types without using them, but it isn't a strong guarantee provided by the type system.

_Unforgettable_ and _undroppable_ types are two different kinds of linear types.

* _Unforgettable types_ are types for which the `Drop` implementation is guaranteed to be called. In other words, it isn't possible to call functions like [`std::mem::forget`](https://doc.rust-lang.org/std/mem/fn.forget.html) or [`Box::leak`](https://doc.rust-lang.org/std/boxed/struct.Box.html#method.leak) on these types.
* _Undroppable types_ are types that cannot be arbitrarily dropped. Instead, these types can only be dropped in some kind of controlled fashion, enforced through encapsulation. (Think of undroppable types are those where there isn't a zero-argument `impl Drop`, or in other words where the `Drop` impl must be passed in an argument.)

Unforgettable types would address many of the issues brought up in the talk, while undroppable types would address all of them. But they're both quite complex to fit into Rust. For more discussion, see [this post by withoutboats](https://without.boats/blog/asynchronous-clean-up/).
