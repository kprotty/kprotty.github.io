---
title: "Sync Primitives are Functionally Complete"
---

You can use any thread synchronization primitive to build any other one. Here's how:

### Reducing into Event

First, the primitive needs to support blocking the caller until some condition occurs. With it, you can then create a _new_ primitive called an `Event`. This is simply a type where one thread calls `event.wait()` which suspends its caller until another thread calls `event.set()`. Think of it as waiting for a bool to become true.

For example, here's how to do it with POSIX threading primitives: With a `Mutex` and `Condvar` it's pretty straight forward; Wrap a bool in the mutex, using `cond.wait()` to wait until it's set and calling `cond.signal()` when setting it. If there's _only_ a `Mutex` available, locking it prematurely turns it into a `Semaphore` where a subsequent `lock()` is `event.wait()` and `unlock()` is `event.set()`. `RwLock` is effectively a `Mutex` here to the same effect.

For a more practical example, here's how to build it from OS threading primitives: Most systems provide a `Futex` API, where `wait(&atomic_int, value)` blocks until the atomic no longer matches the value and `wake(&atomic_int, N)` unblocks N threads waiting on the atomic (after its value has presumably been updated). Some OS like Windows and NetBSD instead provide Events directly, with `NtWaitForAlertByThreadId/NtAlerThreadByThreadId` and `lwp_park/lwp_unpark` respectively.

For a threading primitive where the `set()` caller must block until a matching `wait()` (e.g. [`NtKeyedEvents`](https://locklessinc.com/articles/keyed_events/) or Rust's [`Barrier`](https://doc.rust-lang.org/std/sync/struct.Barrier.html)), it can be made non-blocking by introducing an atomic bool in front; Both `wait()` and `set()` atomic swap the bool to true. If `wait()` sees _False_, set() is yet to arrive so it blocks on the primitive. If `set()` sees _True_, wait() is/will-be blocking so it unblocks the primitive.

### Expanding from Event

Now with an `Event`, it can be used to make any other sync primitive. Let's start with a Mutex:

<details>
<summary>Pseudo-code for a simplistic, Event based Mutex</summary>

```rs
type Node:
    next: ?*Node
    event: Event

type Mutex(state: Atomic(usize) = 0):
    lock():
        s = state.load(Relaxed)
        loop:
            while s & 1 == 0:
                s = state.cas(s, s | 1, Acquire, Relaxed) orelse return
            node = Node{ next: ?*Node(s & ~1), event: Event{} }
            s = state.cas(s, usize(&node) | 1, Release, Relaxed) orelse:
                node.event.wait()
                s = state.load(Relaxed)
                continue
    
    unlock():
        s = state.cas(1, 0, Release, Acquire) orelse return
        loop:
            node = *Node(s & ~1)
            s = state.cas(s, usize(node.next), Release, Acquire) orelse:
                node.event.set()
                return
```

</details>

If curious how this works, check out my previous post on [Building a Tiny Mutex](https://kprotty.me/2022/09/19/building-a-tiny-mutex.html#queued-locks). With just an `Event` (+ optionally a `Mutex`), a `Futex` API can be built [using similar tricks](https://github.com/ziglang/zig/blob/eb375525366ba51c3f626cf9b27d97fc81e2c938/lib/std/Thread/Futex.zig#L506). From `Futex`, all other primitives can be made (as seen on linux [glibc](https://codebrowser.dev/glibc/glibc/nptl/pthread_mutex_lock.c.html) & [musl](https://git.musl-libc.org/cgit/musl/tree/src/thread/pthread_mutex_timedlock.c?h=v1.2.5&id=0784374d561435f7c787a555aeab8ede699ed298)). Here's some examples:

<details>
<summary>Pseudo-code for simplistic, Futex based POSIX primitives</summary>

```rs
type Mutex(state: Atomic(u32)):
    lock():
        _ = state.cas(0, 1, Acquire, Relaxed) orelse return
        while state.swap(2, Acquire) != 0:
            Futex.wait(&state, 2)
    unlock():
        if state.swap(0, Release) == 2:
            Futex.wake(&state, 1)

type Condvar(state: Atomic(u32)):
    wait(mutex):
        s = state.load(Relaxed)
        mutex.unlock()
        Futex.wait(&state, s)
        mutex.lock()
    signal(): _wake(1)
    broadcast(): _wake(max(u32))
    _wake(n):
        _ = state.fetchAdd(1, Release)
        Futex.wake(&state, n)
```
</details>


It's all pretty neat, but does it make sense to do this in practice? Yes.

Golang implements all its blocking using an `Event` within each goroutine, where `wait()` is [`gopark`](https://github.com/golang/go/blob/a4d99770c0e5f340d6d11d6353110413dc109138/src/runtime/proc.go#L443) and `set()` is [`goready`](https://github.com/golang/go/blob/a4d99770c0e5f340d6d11d6353110413dc109138/src/runtime/proc.go#L479). A `Futex`-style [implementation](https://github.com/golang/go/blob/release-branch.go1.25/src/runtime/sema.go) (exposed as a semaphore) is then written using said goroutine `Event`, and the other sync primitives then use that semaphore API.

Also, a while back I wrote a Rust crate called [µSync](https://github.com/kprotty/usync) which implements all Rust standard library (stdlib) sync primitives using an `Event` based on [`std::thread::park()`](https://doc.rust-lang.org/std/thread/fn.park.html). However, unlike Golang, this skips having an intermediary `Futex` and instead builds each directly on top of `Event`. The benchmarks showed it matching or outperforming those in the stdlib at the time, and some of the strategies wounded up [contributed upstream](https://github.com/rust-lang/rust/pull/110211) (thanks joboet!).

So with one you can always make the others. The more interesting question is whether that's a good idea..

I think so, at least.