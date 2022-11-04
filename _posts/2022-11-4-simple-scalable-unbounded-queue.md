The thing I love most about programming over the years has always been design and optimization work. This is what got me into compilers, garbage collectors, runtimes, schedulers, and eventually [databases](https://twitter.com/kingprotty/status/1567602181074829317). Nothing gets me more excited than tag-teaming with someone to research, test, benchmark, and go down various rabbit holes with the goal of making something cool. In this case, I've decided to share one of the results!

## Context

For those uninitiated, a "channel" refers to a type of queue generally used in a programming language for passing data from one concurrent task to another with extra amenities. At least, this is how I describe it. I'll be using the term interchangeably with "queue" from now on. There's a few different properties of channels that help categorize them for use and comparisons: 

**Bounded or Unbounded**. This describes whether or not the channel has a maximum capacity of stuff it can have in it which hasn't been dequeued. Channels with a bound will either be **Blocking or Non-Blocking**. Blocking channels will pause the caller until it can interact with the channel whereas non-blocking channels will return immediately with some sort of error.

Finally, there's the amount of concurrent **Producers and Consumers** that are allowed. A channel which allows multiple enqueues to happen at the same time would be considered *Multi-Producer*. One which doesn't would be considered *Single-Producer*. In this post, I'll be specifically talking about writing an **Unbounded, Non-Blocking, MPSC** (*Multi-Producer Single-Consumer*) channel.

You can find these types of channels anywhere data flows like a sink: multiple threads sending data to a single thread. [Actor Mailboxes](https://actoromicon.rs/ch03-00-actors.html#a-mailbox) often use such an implementation. [Serial Queues](https://developer.apple.com/library/archive/documentation/General/Conceptual/ConcurrencyProgrammingGuide/OperationQueues/OperationQueues.html) in Apple's GCD (`libdispatch`) also use something like this. The one relevant to our story though is the mpsc provided by the Rust standard library.

## Foundation

There's been some talk in the Rust community about channel performance and it piqued my interest. A while back, [`ibraheemdev` and I](https://twitter.com/ibraheemdev/status/1543813995848781824) went crazy with experimenting and researching to find optimal channel implementations. There were [a lot](https://github.com/kprotty/jiffy/branches) of algorithms tested, but he stumbled upon something interesting called [loo-queue](https://github.com/oliver-giersch/looqueue).

Oliver Giersch and Jörg Nolte published [a paper](https://github.com/nim-works/loony/blob/main/papers/GierschEtAl.pdf) (along with a C++ and Rust implementation, bless them) called *Fast and Portable Concurrent FIFO Queues With Deterministic Memory Reclamation"* [[ref](https://ieeexplore.ieee.org/abstract/document/9490347)]. In this, they note that `fetch_add` (atomic "fetch and add", or **FAA**) scales better than looping with `compare_exchange` (atomic "compare and swap" or **CAS**) on x86. You can verify this yourself with a simple [zig benchmark](https://gist.github.com/kprotty/9f45dde0eaea94a9a8d13097ee44b3cf).

Their idea was remarkably straight-forward: Use FAA on the producer to get the current buffer while also reserving an index into it atomically. If the reserved index is outside the buffer's bounds, then install a new buffer to the producer. It's the most ideal starting point for a concurrent queue but it has a few edge case that needs to be addressed.

## Encoding

First, we need a way to pack the buffer pointer along with the index into the same atomic variable so FAA can be used. Assuming an atomic variable is at most `usize` large, we'll need to fit that all in there using some form of [pointer tagging](https://en.wikipedia.org/wiki/Tagged_pointer).

Since this is focused on x86_64 (as that's where FAA is actually scalable), we have two options for tagging the buffer pointer. The first option is to [align the buffer pointer](https://stackoverflow.com/questions/4322926/what-exactly-is-an-aligned-pointer) to the amount of items it can hold (its size) guaranteeing that its address will have the bottom bits zeroed out to store the index: `[pointer_bits:u{64-N}, index_bits:uN]: u64`. This only works if our buffer size is a power of two, which we would've done anyway.

The other option is to take advantage of a special property on x86_64 called [canonical addresses](https://en.wikipedia.org/wiki/X86-64#Canonical_form_addresses). Although pointers are 64 bits in size, they only use a portion of it for addressing memory. In traditional 4-level paging for userspace, [only the first 48 bits are used]. The remaining 16 bits must just be a copy of that 48th bit (using 1-based indexing here) when trying to dereference it. Instead of having to align the buffer when allocating, we could pack the index into these unused bits.

One could leave it at that and continue on implementing the algorithm but there's a subtle bug here: FAA is an unconditional atomic operation. This means that it could increment the index bits even when it's already at the max value and overflow into the buffer's pointer bits. Worst of all, only the thread which did the fatal increment would know, leaving others to incorrectly race on indexes already reserved as it wraps around.

For such a queue to work, the index bits need to have enough overshoot room to contain indexes larger than the buffer size and enough to handle the max amount of concurrent producer threads. So, for example, if the buffer size is `256` and the index bits are aligned to `512` (larger than `256`), then that allows at most `512 - 256 - 1(corruptor) = 255` concurrent producers seeing a full buffer before state becomes corrupted. Having a bound on max producers is *meh*, but if you even have that many hitting the same queue, you should reconsider your program's design... 

## Extending

So the fast path is clear now: 1. `FAA(producer, 1)` 2. If it reserved a valid buffer and index, write to that slot and make it available for the consumer. Now we need to define the slow path, as this is what makes or breaks the scalability of a concurrent algorithm.

When a producer observes that all buffer slots have been reserved to be written to, it needs to allocate a new buffer and replace the existing one. Since there could be multiple producers that observe this state, we have a few options of how to install a new buffer:

**Option 1.** Wait for the first producer which sees full to install a new one. This means that every "overflowing" producer which didn't see `index = 256` just waits until the producer's buffer changes, then tries to FAA again. This can reduce contention (less threads updating the producer) but it means that producers can now block which isn't ideal for what is supposed to be a lock-free algorithm..

**Option 2.** Everyone allocates their own new buffer and tries to install it to the producer. This eliminates the chance of a producer blocking but can increase memory pressure as at most `255` producers could potentially allocate their own buffer and `254` would have to free it when one wins the install race. We can do better than that..

**Option 3.** Someone installs the new buffer somewhere else and everyone helps update the producer with it. This solves a bunch of stuff: It doesn't block like before and having a relatively uncontended place that can be observed before deciding to allocate a buffer means less memory pressure for the consumer. Everyone helping to install the same buffer also means that less producers will see the overflow state.

## Merging

We can actually kill two birds with one stone here too. The *"somewhere else"* can refer to the previous buffer's `.next` pointer. If there is no previous buffer (i.e. it's one of the first enqueues), the "next pointer" can refer to the consumer (who will be waiting for a non-null buffer before reading the slots). This implements *Option 3* while also linking the previous buffer to the new one for the consumer to continue reading slots.

Another trick I learned from [`crossbeam`](https://crates.io/crates/crossbeam-channel) (a popular Rust crate which has a "fast" channel) is that buffer allocation can be amortized. Instead of always allocating and deallocating a buffer if we fail to install it to the previous buffer link, we can keep it in a local variable. If a producer is unluckly enough to see the overflow condition multiple times, it can reuse the buffer it allocated last time for linking. If it manages to link its allocated buffer, it sets the local variable to null, and once a slot is reserved it frees the local variable if its not null.

Finally, when we're updating the producer with the new buffer, we can reserve a slot at the same time. Just update it with the index as 1 and write to slot 0. It acts as doing a FAA right after without the unnecessary contention. Our implementation is starting to look really optimized! Too bad it's still incomplete..

## Collecting

If you're implementing this in a garbage collected language, you can stop here. Alas, many languages looking for high performance queue implementations in the first place don't have that luxury (and for good reason). Now we have to address the aspect that kills most concurrent data structure ideas: concurrent memory reclamation.

In this case, the algorithm has been designed to account for this in a clever way. If you squint tightly, you'll notice that the FAA to reserve an index **also acts as a reference count** for the amount of threads that have seen the block in the overflow condition (`index - buffer size`). We can use this property to know when to free the potentially full old buffer.

We introduce a counter on the buffer called `.pending`. If a thread loses the race to install the new buffer after observing the overflow condition, it will decrement the old buffer's pending count by 1 before retrying to FAA again. Also, when the consumer finds a new linked buffer, it too will decrement the pending count of the previous buffer.

The thread which wins the race for installing the new buffer will know how many other threads saw/accessed the old buffer from the overflowed index. It will then *increment* (the opposite of others) that amount from the old buffer's `.pending`. This will cancel out the decrements from the losing producers as well as the consumer which represents the winner's count. Any thread which changes `.pending` to 0 will be the one to free the buffer.

One final edge case is when there is no *"old buffer"*. This happens for the first enqueue by the producers. Here, the losing producers just retry FAA without doing anything and the winning producer increments the `.pending` count with 1 instead to account for and cancel out the consumer doing the same when it finds the next buffer.

I cannot overstate how simple and effective of an idea this is. A single atomic RMW operation is used to both reserve a slot to write *AND* to increment a reference count to track active users. In the paper, their algorithm is actually *multi-consumer* so it does a few more tricks to make concurrent reclamation work but the idea is the same.

Anyways, here's the final algorithm:

```rs
type Slot(T):
    value: Uninit(T)
    ready: Atomic(bool)

    read() -> ?T:
        if not LOAD(&ready, Acquire): return null
        return READ(&value)

    write(v: T):
        WRITE(&value, v)
        STORE(&ready, true, Release)

@align(4096)
type Buffer(T):
    slots: [buffer_size]Slot(T)
    next: Atomic(?*Buffer(T))
    pending: Atomic(isize)

    // basic refcount stuff
    unref(count: isize):
        p = ADD(&pending, count, Release)
        if (p + count != 0) return

        FENCE(Acquire)
        free(this)

type Queue(T):
    producer: Atomic(?*Buffer(T))
    consumer: Atomic(?*Buffer(T))

    push(value: T):
        cached_buf = null
        defer if (cached_buf != null) free(cached_buf)

        loop:
            // fast path
            (buf, idx) = decode(ADD(&producer, 1, Acquire))
            if (buf != null) and (idx < buffer_size):
                return buf.slots[idx].write(value)

            // find where to register & link next buffer
            prev_link = if (buf != null) &buf.next else &consumer
            next = LOAD(prev_link, Acquire)

            if (next == null):
                // cache the malloc
                if (cached_buf == null) cached_buf = malloc(Buffer(T))
                next = cached_buf

                match CAS(prev_link, null, next, Release, Acquire):
                    Ok(_): cached_buf = null // registered so dont free it
                    Err(updated): next = updated
            
            p = LOAD(&producer, Relaxed)
            (cur_buf, cur_idx) = decode(p)
            loop:
                // retry FAA if failed to install
                if (buf != cur_buf):
                    if (buf != null) buf.unref(-1)
                    break

                // install new buffer + reserve slot 0 in it
                if Err(updated) = CAS(&producer, p, encode(next, 1), Release, Relaxed):
                    p = updated
                    continue

                (old_buf, inc) = (buf, cur_idx - buffer_size)
                if (buf == null):
                    (old_buf, inc) = (next, 1) // account for consumer

                old_buf.unref(inc)
                return next.slots[0].write(value)

    pop() -> ?T:
        (buf, idx) = decode(LOAD(&consumer, Acquire))
        if (buf == bull): return null

        if (idx == buffer_size):
            next = LOAD(&buf.next, Acquire)
            if (next == null): return null

            buf.unref(-1)
            (buf, idx) = (next, 0)
            STORE(&consumer, encode(buf, idx), Unordered)

        value = buf.slots[idx].read()
        if (value != null):
            STORE(&consumer, encode(buf, idx + 1), Unordered)
        return value
```

## Specializing

I've hyped up this FAA algorithm, but I should note that this doesn't scale that well on ARM. With the rise of Apple M1 chips, `aarch64` as a target architecture is a lot more prevalent. Even with [optimized instructions for FAA](https://developer.arm.com/documentation/dui0801/g/A64-Data-Transfer-Instructions/LDADDA--LDADDAL--LDADD--LDADDL--LDADDAL--LDADD--LDADDL) can be slower than simple CAS based queues. 

I'm not entirely sure why, but I speculate it's because ARM chips don't do as well under various forms of atomic contention compared to x86 chips, even if they excel at single threaded execution. You can see this reflected in various [geekbench/cinebench scores](https://techjourneyman.com/blog/apple-m1-vs-amd-ryzen-9/) online as well as locally (on an M1) using that [FAA vs CAS script](https://gist.github.com/kprotty/9f45dde0eaea94a9a8d13097ee44b3cf). The latter shows that CAS (with backoff via [`wfe`](https://www.dpdk.org/wp-content/uploads/sites/35/2019/10/Armv8.pdf)) scales **5x** better than FAA on M1.

> **Note**: Compiling to baseline aarch64 (< ARM v8.2) without the accelerated [CAS instruction](https://developer.arm.com/documentation/ddi0596/2020-12/Base-Instructions/CAS--CASA--CASAL--CASL--Compare-and-Swap-word-or-doubleword-in-memory-) still has FAA as the slowest, but `wfe` backoff is slightly worse than a normal CAS loop. Switching backoff to the standard randomized spinning (with [`isb`](https://bugs.java.com/bugdatabase/view_bug.do?bug_id=8258604) instead) as done on x86 makes it **3x** faster than FAA on M1.

If you can't beat 'em, join 'em. The queue implementation can be specialized just for M1 (or [LL/SC](https://en.wikipedia.org/wiki/Load-link/store-conditional) architectures in general). The idea is the same (pointer tagged buffers with their index), except we use CAS and apply backoff appropriately on failure. If the index isn't at the buffer limit, bump it with CAS to write to that slot. If it's at the limit or the buffer is null (i.e. first enqueue) install a new buffer (using alloc caching from before) with index as 1, write to slot 0, and link new buffer to old.

## Benchmarking

A post about writing scalable algorithms can't be complete without benchmarks. I've implemented the original FAA algorithm and the specialized algorithm then put them up against well known channels in the Rust ecosystem like [`crossbeam`](https://github.com/crossbeam-rs/crossbeam/tree/master/crossbeam-channel), [`flume`](https://github.com/zesterer/flume), and Rust's standard library mpsc.

One method of benchmarking channels is to spawn N producer threads which each send a fixed amount of items, then receive all of the items on the consumer thread. I believe this doesn't measure throughput correctly: Spawn could switch to a producer thread and send its entire amount before spawning the next. Producers could complete sooner than the consumer and now it only benchmarks the consumer. Producers could yield on contention, let the other go through its amount, and finish its amount without contention.

To properly measure contention, the producer and consumer run for a fixed amount of time (one second at the moment) instead of for a fixed amount of items. This allows producers to contend with each other. Couple this with a [`Barrier`](https://doc.rust-lang.org/std/sync/struct.Barrier.html) to ensure everyone is running only when they've all been spawned, producers can race with the consumer and possibly have the latter out-run the former and hit the slow/blocking path.

Also, when I write benchmarks for synchronization primitives, I like to make sure it's measuring various properties of it and not just simple throughput. In this case, each implementation runs for one second and collects metrics that provide insight to not just a channel's throughput, but also its scheduling fairness, ingress rate, and producer latency:

* **recv**: The amount of items received by the consumer. Records consumer throughput.
* **sent**: The amount of items sent by all producers. Records producer throughput and can be divided by **recv** to get ingest rate.
* **stdev**: Standard deviation of enqueues between producers. Higher implies unfair producer scheduling and potentially producer starvation.
* **min & max**: The minimum/maximum about of sends done by any producer thread. Helps concretely conceptualize unfairness.
* **<50% & <99%**: Latency percentiles in elapsed time for all enqueues done by all producers. p50 implies how fast most enqueues were while p99 implies how slow enqueues may have become.

We're not through with trying to be thorough! Channel benchmarks will also only run their tests in a simple setup on their machine. This could mean `${num_cpus}` producers or even some fixed amount regardless of the system.. Instead, each channel implementation has the metrics collected in a set of different scheduling environments. The variety shows how they perform not just in ideal cases but also extreme ones:

* **spsc**: One producer and one consumer. The simplest setup to benchmark ingest rate.
* **micro contention**: Same as before, but now two producers who may fight over each other.
* **traditional contention**: About 4 producers. This is on average the most contention channel usage in practice will see, at least speculatively.
* **high contention**: `${num_cpus}-1` producers and one consumer. Each logical cpu core should be busy doing something and this implies max _hardware_ scalability.
* **over subscribed**: `(2 * ${num_cpus}) - 1` producers. The scheduler now has more threads than logical cpu cores and has to balance them out. Unfair scheduling policies start to suffer here.
* **busy system**: Same as _high contention_, but there are `${num_cpu}` unrelated threads pinned to logical cores and hogging up their execution time. Scheduler now has to decide which is more important: producers, consumers, or the busy threads. Unbounded yielding policies start to suffer here.

## Disconnecting

Thanks for reading all the way through and totally not skipping to the end (👀 right?). Anyways, I've published this channel implementation, as well as the benchmarks, in a Rust crate called [`uchan`](https://crates.io/crates/uchan). Check out my other crates as well: [`usync`](https://crates.io/crates/usync) for word-sized sync primitives and [`uasync`](https://crates.io/crates/uasync) for an `unsafe`-free, dependency-free, async runtime.

Oh yea, I have a [Ko-Fi](https://ko-fi.com/kprotty) now so, if you're interested, you can buy me a ~~coffee~~ ~~beer~~ cloud VPS instance to run more benchmarks. Otherwise, you can find me on [twitter](https://twitter.com/kingprotty) or in the Rust/Zig community discords/reddits. I still haven't covered _bounded_ channels yet (and I gots some ideas brewing) so stay tuned.
