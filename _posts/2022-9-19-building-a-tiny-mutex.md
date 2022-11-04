---
title: "Building a Tiny Mutex"
---

Lets say you're writing your own synchronization primitives for whatever reason and you need a small, fast Mutex. 

Could be that you made your own OS and pthread's API ain't looking too good. Could be that you want something faster than what your platform's libc provides through pthread. Could be that you're doing it for fun (the best reason), it doesn't matter.

In a perfect world, you just write a simple spin-lock and be done with it. But scheduling isn't that easy and such naïve solutions can have pretty bad or awkward consequences. In this post I'll walk through designing and understanding what makes a good mutex. I'll assume you know some about atomic memory operations and their memory orderings (pretty thicc assumption, I know).

## Spin Locks

You love to see 'em. First, let's start off with that spin-lock idea from before:
```rs
// Excuse the pseudo-code looking zig hybrid.
// CAS() = `compareAndSwap() ?T` which returns null on success and the current value on failure

locked: bool = false

lock():
    while CAS(&locked, false, true, Acquire, Relaxed) != null:
        YIELD() // assume this is _mm_pause() instead of sched_yield(). Almost never do the latter.

unlock():
    STORE(&locked, false, Release)
```
I'm not going to go into atomics and their orderings, but this is a basic spin-lock. For those who write something like this unironically, I'm glad you're reading. For those who noticed that I should be spinning with `LOAD` instead of CAS, *the bottom of this post ~~is~~ was for you (see [conclusion](#closing-notes))*. For those who didn't understand or realize why, there's an opportunity to learn some stuff here.

So we know that a spin-lock tries to switch the `locked` flag from `false` to `true` and spins until it can, but what do the threads look like to the machine when this happens? Each thread is continuously doing a `CAS` hoping that it will acquire the Mutex. On x86, this is a `lock cmpxchg` instruction which unfortunately acts a lot like `lock inc` or `lock xadd` as seen in reference counting. Why is this unfortunate? Well, we need to dig into how caching works.

### Cache Coherency

Each CPU core on modern systems abstractly has its own cache / fast-access view of memory. When you do an operation that reads or writes to memory, it happens on the cache and that needs a way to communicate these local changes to other CPU core caches and to main memory. This process is generally referred to as "cache coherency"; the dance of maintaining a coherent view of memory across caches.

A nice protocol which explains this is [M.E.S.I.](https://en.wikipedia.org/wiki/MESI_protocol). You can read more about it if you want, but I just want to touch on some of it for this to make sense. Basically, caches work with (generally, 64 byte) chunks of memory called "lines". CPU cores send messages to each other to communicate the state of lines in caches. Here's an ***extremely simplified*** example:

* CPU-1 (C1) reads the line from main memory into their cache and tells everyone else. No one else has it in their cache so it stores the line as `Exclusive` (E).
* C2 does the same and gets told that C1 also has the line. Now both C1 and C2 store the line as `Shared` (S).
* C1 writes to the line and updates its local line state to `Modified` (M). It then (atomically) tells others about this (C2) which update their view of the line to `Invalid` (I)
* C2 tries to read the line but it's now Invalid instead of Shared. C2 must now wait for C1 to stop modifying then re-fetch the line from main memory again as Shared (ignore that snooping exists plz).
* C1 writes the new line value to main memory and moves its view of the line from Modified to Shared again. Others (C2) can now read the new value from main memory as Shared.

When one CPU core is the only one interacting with a line, reads and writes to it are basically free. It can just update its local cache and keep the line as `Modified` for as long as it wants. The problem comes when other CPU cores wanna take a peek while its writing. If the original core is not writing, then everyone can read from their local cache seeing `Shared` with no overhead. When even one core writes, other caches need to be `Invalid`ated while waiting for the writer to flush to main memory. This expensive "write-back" process is known as "contention".

### Contention

Let's go back to the spin-lock's `lock cmpxchg` from earlier. This x86 instruction, along with the others previously listed, are knows as *read-modify-write* (RMW) atomics; The last bit being the most important. CPUs that are waiting for the mutex holder to unlock are continuously doing CAS and writing to the same line. This generates a lot of needless contention by invaliding the lines from other cores, making them wait for this unnecessary write to re-fetch from main memory only to see the mutex still locked and repeat. 

Instead, [AMD recommends](https://gpuopen.com/gdc-presentations/2019/gdc-2019-s2-amd-ryzen-processor-software-optimization.pdf) (slide 46) that you should spin by `LOAD`ing the line instead of `CAS`'ing it. That way, waiting cores only invalidate other's caches when the mutex is unlocked and can be acquired. Each waiting core still pays the cost of re-fetching from main memory once it changes, but at least they only re-fetch when the mutex is unlocked or if a new core is just starting to lock. 

```rs
try_lock():
    return CAS_STRONG(&locked, false, true, Acquire, Relaxed) == null

lock():
    // Assume the mutex is unlocked. Proper mutex usage means this should be the average case
    if try_lock(): return

    do: 
        while not LOAD(&locked, Relaxed):
            YIELD()
    while not try_lock()
```

### Unbounded Spinning

But remember that we're designing a userspace Mutex here. And in userspace, we deal in threads not CPU cores. Many threads are queued up to run on a smaller amount of cores so just spinning like that can be pretty bad. There's a few reasons why you shouldn't use this sort of spin-lock, whether you're in userspace or even the kernel.

Kernel spin-locks at the CPU core level generally do more than just spin. They may also disable hardware interrupts to avoid the spinning code being switched to an interrupt handler. If you're at the kernel level, you can also put the core to a deeper sleep state or choose to do other stuff while you're spinning. AFAIK, kernel spin-locks also prefer explicit queueing over single bools to avoid multiple cores re-fetching the line.

Userspace spin-locks suffer from accidental blocking. If the mutex lock owner is de-scheduled, the waiting threads will spin for their entire quota, never knowing that they themselves are preventing the lock owner from running on their core to actually unlock the mutex. You could say *"have `YIELD()` just reschedule the waiting thread"* but this assumes that 1) the lock owner is scheduled to the same core for rescheduling to give it a chance 2) that `YIELD()` will reach across cores to steal runnable threads if the lock owner isn't locally queued to run and 3) that `YIELD()` will actually yield to your desired thread. 

The first isn't always true due to, uh oh, 2022 high core counts and sophisticated/non-deterministic OS scheduling heuristics. The second isn't currently true for Linux [`sched_yield`](https://elixir.bootlin.com/linux/latest/source/kernel/sched/core.c#L8257) or Windows' [`SwitchToThread`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-switchtothread). The third doesn't really fly as your program is likely sharing the entire machine with other programs or virtual machines in these times. Linus Torvalds goes into more passionate detail [here](https://www.realworldtech.com/forum/?threadid=189711&curpostid=189752).

Don't get me wrong, spinning *can* be a good thing for a Mutex when it comes to latency. If the lock owner releases it "soon" while there's someone spinning, they can acquire the mutex without having to go to sleep and get woken up (which are relatively expensive operations). The problem is "spinning for too long" or, in our worst case, spinning theoretically indefinitely / without bound.

What we want instead is a best of both worlds; Spin for a little bit assuming we can minimize acquire latency. Then, if that doesn't work, queue the thread onto the Mutex so the OS can schedule other threads (possibly the lock owner) on our CPU core. This is called ["adaptive spinning"](https://lwn.net/Articles/314512/) in the literature. Sketching that out, it would look something like this:

```rs
lock():
    if try_lock(): return

    for i in 0..SPIN_BOUND:
        YIELD()
        if try_lock(): return

    do:
        queue_thread_and_block_if_locked()
    while not try_lock()

unlock():
    STORE(&locked, false, Release)
    dequeue_thread_and_unblock_if_any()
```

## Queued Locks

Enter queued locks, or "making the implicit thread queueing explicit". These type of locks represent each waiting task (whether it's a thread in userspace or a core in the kernel) as a linked list node waiting for the mutex to be unlocked. Why a linked lists? Well, having to dynamically heap allocate arrays for a Mutex is kind of cringe. Also, managing such array buffers would require concurrency reclamation (read: GC) and synchronized access (read: [Yo dawg](https://knowyourmeme.com/memes/xzibit-yo-dawg), I heard you like locks. So I put a lock in your lock). Besides heap allocation, linked lists are also a good choice for representing unbounded, lock-free data structures. We'll use a [Treiber stack](https://en.wikipedia.org/wiki/Treiber_stack) for the thread queue.

We also need a way to put the thread to sleep and wake it up. This part relies fully on the platform (OS/runtime) we're running on. The abstraction we can use is an [`Event`](https://en.wikipedia.org/wiki/Event_(synchronization_primitive).) where the waiting thread calls `wait()` and the notifying thread calls `set()`. The waiter blocks until the event is set, returning early if it was already set. It only needs have a single-producer-single-consumer (SPSC) relationship as we'll see in a moment. There's various ways to implement the `Event`:

- On Linux, we can use a local 32-bit integer + [`futex`](https://man7.org/linux/man-pages/man2/futex.2.html).
- On OpenBSD, FreeBSD, DragonflyBSD we can used the ~~scuffed~~ futex apis [`futex`](https://man.openbsd.org/futex.2), [`_umtx_op`](https://www.freebsd.org/cgi/man.cgi?query=_umtx_op&sektion=2&n=1), and [`umtx_sleep`](https://man.dragonflybsd.org/?command=umtx&section=2) respectively.
- On NetBSD, we can use [`lwp_park`](https://man.netbsd.org/_lwp_park.2)/[`lwp_unpark`](https://man.netbsd.org/_lwp_unpark.2) which are really nice for single-thread wait/wake mechanisms.
- On Windows, we *could* use [`WaitOnAddress`](https://docs.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-waitonaddress) but we're cool, fast, and instead use the undocumented (but documented by the entire internet) [`NtWaitForAlertByThreadId`](https://docs.rs/ntapi/latest/ntapi/ntpsapi/fn.NtWaitForAlertByThreadId.html)/[`NtAlertThreadByThreadId`](https://docs.rs/ntapi/latest/ntapi/ntpsapi/fn.NtAlertThreadByThreadId.html) that WaitOnAddress calls internally anyway.
- On pretty much everywhere else, the sync primitives kind of suck and were stuck making a binary semaphore using [`pthread_mutex_t`](https://pubs.opengroup.org/onlinepubs/007908799/xsh/pthread_mutex_lock.html)/[`pthread_cond_t`](https://pubs.opengroup.org/onlinepubs/007908799/xsh/pthread_cond_wait.html).
- On Darwin (macOS, iOS, giveUsFutexOS) we could be safe and stick with pthread or be cheeky/fast and use [`__ulock_wait2`](https://github.com/apple/darwin-xnu/blob/main/bsd/sys/ulock.h#L66-L67)/[`__ulock_wake`](https://github.com/apple/darwin-xnu/blob/main/bsd/sys/ulock.h#L68) while risking our program getting rejected from the AppStore for linking to undocumented APIs.

Let's combine this Event primitive with our Treiber Stack to create our first, lock-free, queued Mutex.

```rs
type Node:
    next: ?*Node = null
    event: Event = .{}

type Stack:
    top: ?*Node = null

    push(node: *Node):
        node.next = LOAD(&top, Relaxed)
        loop:
            node.next = CAS(&top, node.next, node, Release, Relaxed) orelse return
    
    pop() ?*Node:
        node = LOAD(&top, Acquire)
        while node |n|:
            node = CAS(&top, node, n.next, Acquire, Acquire) orelse break
        return node

waiters: Stack = .{}

lock():
    ...
    do:
        node = Node{}
        waiters.push(&node)
        node.event.wait()
    while not try_lock()

unlock():
    node = waiters.pop() // stack is single-consumer so pop before unlocking
    STORE(&locked, false, Release)
    if node |n| n.event.set()
```

Great, looks good! But there's an issue here. Remember that I named the queueing/blocking function `queue_thread_and_block_if_locked()`? We're missing that last part (`_if_locked`). A thread could fail the first try_lock(), go queue itself to block, the lock owner unlocks the Mutex while its queueing, then the thread blocks even when the Mutex is unlocked and now we have a [deadlock](https://en.wikipedia.org/wiki/Deadlock). We need to make sure that the queueing of the thread is atomic with respect to the Mutex being locked/unlocked and we can't do that with separate atomic variables in this case so we gotta get clever.

### Word-sized Atomics

If both states need to be atomic, let's just put them in the same atomic variable! The largest and most cross-platform atomic operations work on the machine's pointer/word size (`usize`). So to get this to work, we need to encode both the `waiting` Treiber Stack and the `locked` state in the same `usize`.

One thing to note about memory is that all pointers have what's called an ["alignment"](https://en.wikipedia.org/wiki/Data_structure_alignment). Pointers themselves are canonically numbers which index into memory at the end of the day (I don't care what [strict provenance](https://github.com/rust-lang/rust/issues/95228) has you believe). These "numbers" tend to be a multiples of some power of two that's dictated by their `type` in source code or the accesses they need to perform. This power of two multiple is known as alignment. `0b1001` is aligned to 1 byte while `0b0010` is aligned to 2 bytes (read: there's N-1 `0` bits to the right of the farthest/lowest bit).

We can take advantage of this to squish together our states. If we designate the first/0th bit of the `usize` to represent the `locked` boolean, and have everything else represent the stack top Node pointer, this could work. We must just ensure that the Node's address is aligned to at least 2 bytes (so that the last bit in a Node's address is always 0 for the locked bit). 

Queueing and locking have now been combined into the same atomic step. When we unlock, we can choose to dequeue any amount of waiters we want before unlocking. This gives the guarantee to `Event` that only one thread (the lock holder trying to unlock) can call `Event.set` to allow SPSC-usage optimizations. Our Mutex now looks like this:

```rs
type Node aligned_to_at_least(2):
    ...

state: usize = 0

const locked_bit = 0b1
const node_mask = ~locked_bit

try_lock():
    s = LOAD(&state, Relaxed)
    while s & locked_bit == 0:
        s = CAS(&state, s, s | locked_bit, Acquire, Relaxed) orelse return true
    return false

lock():
    // fast path: assume mutex is unlocked
    s = CAS(&state, 0, locked_bit, Acquire, Relaxed) orelse return

    // bounded spinning trying to acquire
    for i in 0..SPIN_BOUND:
        YIELD()
        s = LOAD(&state, Relaxed)
        while s & locked_bit == 0:
            s = CAS(&state, s, s | locked_bit, Acquire, Relaxed) orelse return

    loop:
        // try to acquire if unlocked
        while s & locked_bit == 0:
            s = CAS(&state, s, s | locked_bit, Acquire, Relaxed) orelse return 
        
        // try to queue & block if locked (fails if unlocked)
        node = Node{}
        node.next = ?*Node(s & node_mask)
        new_s = usize(&node) | (s & ~node_mask)
        s = CAS(&state, s, new_s, Release, Relaxed) orelse blk:
            node.event.wait()
            break :blk LOAD(&state, Relaxed)

unlock():
    // fast path: assume no waiters
    s = CAS(&state, locked_bit, 0, Release, Acquire) orelse return

    loop:
        top = *Node(s & node_mask)
        new_s = usize(top.next) | (s & ~locked_bit)
        s = CAS(&state, s, new_s, Release, Acquire) orelse:
            return top.event.set()
```

## Locking It Up

At this point, we're basically done. You can now take this Mutex and ship it. After all, this is [what Golang did](https://github.com/golang/go/blob/master/src/runtime/lock_sema.go#L26-L129). But to be real with you, don't use this thing in practice... While it's pretty simple and cross-platform, the OS/runtime developers generally do it better. The big 3 all do special optimizations that I can't hope to ever account for in such an abstract Mutex implementation:

A basic [3-state lock](https://github.com/kprotty/zig-adaptive-lock/blob/blog/locks/futex_lock.zig) on Linux (and the other BSDs that have `futex`) can be faster than what we have here. Not only does it require less updates to the userspace atomic (which means less line contention), but queueing, thread blocking, and the synchronization of that all is handled by the kernel which can do it better than userspace can since it knows which CPU cores / threads are running and all. 

Darwin's `os_unfair_lock` uses thread-IDs instead of queues. Storing the thread-ID means they always know which thread currently holds the mutex lock. By deferring blocking and contention handling to the kernel via `__ulock_wait(UL_UNFAIR_LOCK)`, this allows it to handle [priority inversion](https://en.wikipedia.org/wiki/Priority_inversion), optimized thread queueing, and yielding to the lock owner directly if it was de-scheduled.

Windows' `SRWLOCK` is similar to our implementation but does so much more. It uses a FIFO queue instead of a LIFO stack by intelligently linking the nodes together so it suffers less from unfair waking policies. The kernel also maps a read-only chunk of memory to all user processes called [`KUSER_SHARED_DATA`](https://docs.microsoft.com/en-us/windows-hardware/drivers/ddi/ntddk/ns-ntddk-kuser_shared_data) which exposes useful, kernel-updated stuff like `UnparkedProcessorCount` to know if there's other CPUs running to avoid spinning, `CyclesPerYield` to know the optimal amount of `YIELD()`s, `ProcessorFeatures` to know if the CPU supports spinning with optimized instructions like [`mwait`](https://www.felixcloutier.com/x86/mwait), and much more.

### Closing Notes

So I actually had [a lot more planned](https://github.com/kprotty/zig-adaptive-lock/blob/blog/blog.md#optimizing-spinning-shared-bound) for this post which went deeper into optimizing the Mutex. Before finishing, I [benchmarked](https://github.com/kprotty/zig-adaptive-lock/tree/blog/locks) the fast version and realized it wasn't as good as I led on. This post ended up being under half of what it could've been... If you're still curious, I also published a Rust crate called [usync](https://lib.rs/crates/usync) which implements a bunch of word-sized synchronization primitives including the fast version of this Mutex. Feel free to port the code to whatever language you prefer and/or plug in your own `Event` type to support your platform. 

P.S. I tried out a new writing style this time! My previous posts were focused more about packing as much information as possible. This time, it was similar but I added some personality (may or may not be a good thing). Let me know if you preferred this style or not.