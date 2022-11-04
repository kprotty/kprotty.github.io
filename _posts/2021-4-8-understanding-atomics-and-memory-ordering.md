---
title: "Understandnig Atomics and Memory Ordering"
---

Atomics and Memory Ordering always feel like an unapproachable topic. In the sea of poor explanations, I wish to add another by describing how I reason about all of this mess. This is only my understanding so if you need a better/formal explanation, I recommend reading through the memory model for your given programming language. In this case, it would be the C11 Memory Model described at [cppreference.com](https://en.cppreference.com/w/cpp/atomic/memory_order). 

# Shared Memory
Software and hardware is getting closer to the limits of performance when it comes to single-threaded execution of code. In order to continue scaling compute performance, a popular solution is to introduce multiple single-threaded execution units - or multi-threading. This form of computation manifests itself at different abstraction levels from multiple cores in a CPU to multiple CPUs in a machine and even multiple machines across a network. This post will be focusing more on cores in a CPU, referring to them as “threads”.

For some workloads, the tasks can be divided cleanly and split off to the threads for execution. Such tasks are known as [embarrassingly parallel](https://en.wikipedia.org/wiki/Embarrassingly_parallel) and need not communicate with each other. This is the ideal that multithreaded algorithms should strive for since it takes advantage of all the existing optimizations available for single-threaded execution. However this isn't always possible and it's sometimes necessary for tasks to communicate and coordinate with each other, which is why we need to share memory between threads.

Communication is hard when your code is running in a preemptive scheduling setting. Such an environment means that, at any point, your code can be interrupted in order for other code to run. In applications, the operating system kernel can decide to switch from running your program to run another. In the kernel, hardware can switch from running kernel code to running interrupt handler code. Switching tasks around like this is known as concurrency and in order to synchronize/communicate, we need a way to exclude that concurrency for a small time frame or we risk operating with incomplete/partial data.

# Atomics
Fortunately, CPUs supply software with special instructions to operate on shared memory which can't be interrupted. These are known as atomic memory operations and fit into three categories: Loads, Stores, and ReadModifyWrites (RMW). The first two are self explanatory. RMW is also pretty descriptive: it allows you to load data from memory, operate on the data, and store the result back into memory - all atomically. You may know RMW operations as atomic [increment](https://docs.microsoft.com/en-us/windows/win32/api/winnt/nf-winnt-interlockedincrement), [swap](https://en.cppreference.com/w/cpp/atomic/atomic/exchange), or [compare and swap](https://en.wikipedia.org/wiki/Compare-and-swap).

To do something "atomically" means that it must happen (or be observed to happen) in its entirety or not at all. This implies that it cannot be interrupted. When something is "atomic", tearing (i.e. partial completion) of the operation cannot be observed. Atomic operations allow us to write code that can work with shared memory in a way that's safe against concurrent interruption.

Another thing about atomics is that they're the only sound (i.e. correctly defined) way to interact with shared memory when there's at least one writer and possibly multiple readers/writers to the shared memory. Trying to do so without atomics is considered a [data race](https://en.wikipedia.org/wiki/Race_condition#Data_race) which is [undefined behavior](https://en.cppreference.com/w/cpp/language/ub) (UB). UB is the act of relying on an assumption outside of your target program model (in our case, the C11 memory model). Doing so is unreliable as the compiler or cpu is allowed to do anything outside of its model.

Data races and the UB it implies isn’t just a theoretical issue. One of the single-threaded optimizations I mentioned earlier involves either the CPU or the compiler caching memory reads and writes. If you don’t use atomic operations, the operation itself could be ellided and replaced with its cached result which could break the logic of your code fairly easily:

```py
# should be an atomic_load() but its data race
while (not load(bool)):
    continue

# a potential single-threaded optimization
cached = load(bool)
while (not cached): # possibly infinite loop!
    continue
```

# Reordering
Atomics solve communication only on atomically accessed memory; but not all memory being communicated can be accessed atomically. CPUs generally expose atomic operations for memory that's at most a few bytes large. Trying to do any other sort of general purpose memory communication means we need a way to make this memory available to threads with other means.

Making memory available to other threads is actually trickier than it sounds. Let's check out this code example:

```py
data = None
has_data = False

# Thread 1
write(&data, "hello")
atomic_store(&has_data, True)

# Thread 2
if atomic_load(&has_data):
    d = read(&data)
    assert(d == "hello")
```

At first glance, this looks like it would work. Even if each thread were preempted between each instruction (line of code here), it seems the assert() should always succeed. Based on my wording, you've probably caught on that this assert() can actually fail! The reason for this is due to another single-threaded optimization called reordering.

Hardware (CPU) or software (the compiler as well) can decide to move around (i.e. “reorder”) your code and instructions any way they please as long as the end result is the same as the source code’s intent. This sort of “instruction scheduling” freedom allows for a variety of optimizations to take place.

One example of reordering is via speculative execution. This is when the CPU starts executing code that hasn't been reached yet, in the opportunistic chance that the results can be ready when that code is eventually reached. This is an amazing single-threaded throughput optimization, but it means that the `atomic_store()` can be started before the `write()`, or the `read()` can be started before the `atomic_load()`; both of which could make the assert() fail.

Another example of reordering is by CPU caches. CPUs don't read/write directly to shared memory since that's [relatively slow](https://gist.github.com/jboner/2841832). Instead, each CPU core has its own fast-access, local memory called cache. Most memory operations are performed on a CPU's cache and eventually flushed to / refreshed from to other caches in a process called [Cache Coherency](https://en.wikipedia.org/wiki/Cache_coherence). In our example, the `atomic_store()` could have flushed from cache to shared memory before the `write()` does (e.g. if flushing is done LIFO), or the `atomic_load()` could refresh in cache before the `read()` is; both of which could cause the assert() to fail.

Even the compiler can reorder instructions, but only those without relationships called *dependencies*. One instruction (line of code) is said to "depend" on a previous instruction if it uses the result from the previous one or if the previous one is a [side effect](https://en.wikipedia.org/wiki/Side_effect_(computer_science)). The compiler is free to reorder instructions anywhere before their dependency, but not after. This means `a = 5; b = 10;` can be reordered to `b = 10; a = 5;` which keeps the same semantics (achieving the same thing) since "a" and "b" don't share a dependency with each other. If it were instead `a = 5; b = a + 1;` then "a" can't be moved after "b" since it wouldn't make logical sense as "b" has a dependency on "a". In our example, `atomic_store()` doesn't have a dependency on `write()` so it can be moved around which can make the assert() fail.

At this point it should be clear that instruction reordering is a thing and, when interacting with shared memory, you have to be aware of it. The problem is that **atomic operations on their own don't prevent reordering**. We need an additional concept for atomics to do this. In C11, atomic operations take in another parameter called "memory ordering" which helps solve this problem.

In our previous code example, there were two main issues: one of reordering and one of visibility. Memory orderings solve them by preventing code from being reordered around atomic operations and ensures that certain data or operations become visible or get conceptually "flushed/reloaded from cache". Lets see what this looks like.

# Release and Acquire
We'll introduce two types of memory orderings for now: [Acquire and Release](https://en.cppreference.com/w/cpp/atomic/memory_order#Release-Acquire_ordering). Release goes on atomic stores and ensures that all memory operations declared before it actually happen before it. Acquire goes on atomic loads and ensures all memory operations declared after actually happen after it. This solves the reordering problem.

We then declare one more constraint: All memory operations before a given Release can be observed to [happen-before](https://en.wikipedia.org/wiki/Happened-before) a matching Acquire. You could think of it as changes from the Release becoming visible in a `git push` manner to the Acquire which does a sort of `git pull`. This solves the visibility problem.

Let's add these to our code example: 

```py
data = None
has_data = False

# Thread 1
write(&data, "hello")
atomic_store(&has_data, True, Release)

# Thread 2
if atomic_load(&has_data, Acquire):
    d = read(&data)
    assert(d == "hello")
```

Note that **Release and Acquire don't do any sort of "waiting" or "blocking" for the data to become ready**. They aren't replacements for known synchronization primitives. Instead, they ensure that *if* our atomic_load() sees `has_data` to be True, *then* it’s also guaranteed to see `write(&data, "hello")` thanks to the matching Acquire and Release barriers so our assert should never fail.

For ReadModifyWrite (RMW) atomic instructions, they can also take in a memory ordering called `AcqRel`. Given RMW operations conceptually do both an atomic load and an atomic store, `AcqRel` makes both operations Acquire and Release respectively. This is useful when you want an atomic operation which both 1) makes memory available to other threads via Release and 2) sees memory made available by other threads via Acquire.

## Fences and Variables
You'll notice that i've been saying *"matching Acquire/Release"*. For our examples, the matching is from the load and store using the same "atomic variable' (`&has_data`). Release and Acquires on different atomic variables don't synchronize with each other, it has to be the same atomic variable.

There's an exception to the rule which manifests itself as fences. Fences are a way to establish memory orderings of normal and atomic memory operations without necessarily associating with one given memory op.

Fences are a bit tricky for me as I have a hard time describing them, but they essentially create the happens-before relationship to surround atomics in a way that corresponds to the memory ordering being used:

* A `fence(Release)` creates a happens-before relationship with another `fence(Acquire)`
* A `fence(Release)` makes subsequent non-Release atomic stores into Release if they have a matching Acquire atomic load or matching `fence(Acquire)`
* A `fence(Acquire)` makes previous non-Acquire atomic loads into Acquire if they have a matching Release atomic store or matching `fence(Release)`.

Here's an example of how we could substitute the per-operation memory orderings with fences:

```py
data = None
has_data = False

# Thread 1
write(&data, "hello")
fence(Release)
atomic_store(&has_data, True)

# Thread 2
if atomic_load(&has_data):
    fence(Acquire)
    d = read(&data)
    assert(d == "hello")
```

## Case Study: Mutex
You may have also noticed that this section is called "Release and Acquire" instead of "Acquire and Release". This is done intentionally as having Acquire first often construes the happens-before relationship. Instead of thinking about lock(Acquire) and unlock(Release), it should instead be thought about unlock(Release) making critical section changes available to lock(Acquire):

```py
mutex = Mutex()
data = None

# Thread 1 (assume locked)
    data = "hello"
    fence(Release)
    mutex.unlock()

# Thread 2 (assume unlocked)
    mutex.lock()
    fence(Acquire)
    assert(data == "hello")
```

The Release ordering for a mutex only serves to "Release" the changes to the next mutex locker, who "Acquires" the previously released changes by the last mutex unlocker. The canonically backwards relationship better demonstrates the happens-before relationship between Release and Acquire compared to just saying *"lock() acquires and unlock() releases"*.

What we have created here is called a Partial Ordering. It's an ordering between two sets of (memory) operations. The reason it's "partial" is because it orders between sets instead of the individual operations themselves: The operations before a Release don't need to be observed happening in the order they were described for an Acquire, they just need to be observed to have happened *at all*.

# Sequential Consistency

There are cases when you need certain atomic operations to be observed in a given order between each other. What we need now is a Total Ordering. This ensures there's some defined ordering between the operations themselves rather than a set of operations and is what the `SeqCst` memory ordering is used for.

Let's see another code example:

```py
head = 0
tail = 0
buf = [...]

# Thread 1
steal():
    h = atomic_load(&head)
    t = atomic_load(&tail)
    if t > h:
        item = buf[h]
        if atomic_cas(&head, h, h + 1):
            return item
    return None

# Thread 2
pop():
    t = tail
    atomic_store(&tail, t - 1)
    h = atomic_load(&head)
    if t > h + 1: 
        return buf[t - 1]
    if t == h + 1 and atomic_cas(&head, h, t):
        return buf[t - 1]
    atomic_store(&tail, t)
    return None
```

This is code taken from the implementation of a [LIFO Deque by Chase. Lev](https://fzn.fr/readings/ppopp13.pdf). What it does isn't necessarily important but it serves as a nice example when `SeqCst` is actually needed. 

For pop(), we want to ensure that the store to `tail` is observed to happen before the load to `head`. If not, pop() may not see the items removed from steal(). Lets try to apply Acquire and Release to pop():

```py
atomic_store(&tail, t - 1, Release)
h = atomic_load(&head, Acquire)
```

This doesn't exactly do what we want: Release prevents stuff *before* the store() being reordered after, and Acquire prevents stuff *after* the load() being reordered before it. There's no guarantee that the store() and load() *themselves* can't be reordered before/after each other.

```
        other memory operations
    ^       |            
    |       X     store release----
    |                             |
    ----load acquire    X         |
                        |         v
        other memory operations
```

In order to ensure that the atomic store() and load() stay in their declared order we either need an Acquire barrier on the store(), which we can semantically achieve using an RMW operation with AcqRel (`atomic_swap(&tail, t - 1, AcqRel)`), or we need `SeqCst`.

```py
atomic_store(&tail, t - 1, SeqCst)
h = atomic_load(&head, SeqCst)
```

SeqCst does two things here: It acts as a Release for stores / Acquire for loads as before, but it also ensures a total-ordering between all SeqCst operations. The total-ordering ensures that the store will be seen before the load for other totally-ordered operations. Because total-ordering only applies to other SeqCst ops, we need to apply SeqCst to everything that relies on the total-ordering. This includes the atomic load/cas in pop() as well as the atomic loads/cas in steal(). The total-ordering property also extends to `fence(SeqCst)` so we can use those to achieve the same reordering effects:

```py
steal():
  t = atomic_load(&tail)
  fence(SeqCst)
  h = atomic_load(&head)
  ...

pop():
  atomic_store(&tail, t - 1)
  fence(SeqCst)
  h = atomic_load(&head)
```

To be clear, `SeqCst` shouldn't be used to somehow gain Acquire on stores or Release on loads. That can lead to incorrect usage: `store(SeqCst); load(Acquire)` doesn't ensure that the store will not be reordered after the load() since the load() isn't a part of its total-ordering (it's not SeqCst as well).

It should be used instead to enforce a total-ordering between multiple atomic variables and introduce partial ordering (Acquire/Release as before) which together can achieve the same effect. More emphasis that **total ordering only applies to other SeqCst atomic operations** or to surrounding ops in relation to `fence(SeqCst)`. See [this issue](https://github.com/rust-lang/nomicon/issues/166) for more warnings.

# Weak orderings

In most cases you probably don't need total-ordering on operations for multiple atomic variables. Having the requirement for `SeqCst` is pretty rare. In practice, `SeqCst` is unfortunately often overused and a problematic sign that the programmer wasn't sure what memory ordering to use... Anyway, when you don't want total-ordering over different atomic variables and don't need partial ordering, you should reach for the [Relaxed](https://en.cppreference.com/w/cpp/atomic/memory_order#Relaxed_ordering) memory ordering (also known as Monotonic under LLVM).

All this does is ensure a total-order between all atomic operations **to the same atomic variable**. In other words, other memory operations not on the same memory location can be reordered around it. So `store(X); load(Y)` can be reordered around each other but `store(Y); load(Y)` can't.

All other memory orderings (Acquire/Release/AcqRel/SeqCst) inherit the Relaxed property of "single variable total-ordering" and are known to be "stronger" than it. Relaxed is useful for things like counters or generic single-atomic data that you just read, update, and check out. You cannot use this to synchronize other normal *or* atomic memory operations.

There are even cases where you don't need the total-ordering on the same atomic variable itself and just want to perform some memory operation atomically (i.e. to be free of data-races). For this, you would use the LLVM's [Unordered](https://llvm.org/docs/Atomics.html#unordered) memory ordering. The need for this ordering is even more rare than the need for `SeqCst`. Unordered also isn't even present in the C11 memory model (it only gets as "weak" as Relaxed).

# Hardware Quirks

On modern CPU instruction set architectures (ISA), normal memory operations are atomic by default. The upside is that you don't pay a price for Relaxed/Unordered memory orderings or atomic loads/stores vs normal operations. The downside is that data-races don't exist for the ISA so it's harder to know if you have one or not. Fortunately there are tools which can instrument your memory accesses to detect data races like LLVM's [ThreadSanitizer](https://clang.llvm.org/docs/ThreadSanitizer.html) (TSAN).

Certain CPU ISAs are known to have Total-Store-Ordering (TSO). This includes things like x86 and SPARC. Here, upon normal memory operations being atomic, they also get partial ordering for free. This means loads are Acquire by default and stores are Release by default. As before, you get the benefit of Release/Acquire operations having no overhead (besides inhibiting compiler optimizations) but it also has its downsides. In this case, it lets you be pretty free with orderings so your Relaxed code that should be Release/Acquire will work there but break on other architectures making it easy to write code with incorrect memory orderings.

The "other" architectures mentioned are called Weakly-Ordered ISAs. This includes things like ARM, AARCH64, POWERPC, RISCV, MIPS, etc. Here, loads and stores are still atomic by default but they're only Relaxed and you pay prices for Acquire/Release. This means that getting ordering wrong gives you a higher chance of observing incorrect behavior. The weaker default orderings theoretically allow for more reordering opportunities by the CPU but this doesn't appear to matter in practice given how much better modern x86 CPUs are for cross-core communication in the general case.

When it comes to Sequential Consistency however, there aren't really any platforms where you get this for free. `fence(SeqCst)` in particular is generally the most costly since it often requires a full barrier to implement which prevents all forms of reordering. On x86, it's achieved with `mfence` although [it can be done cheaper](https://stackoverflow.com/a/50279772) using `lock` prefixed instructions if you're not synchronizing write-combined memory instructions. SeqCst loads/stores often require either promotion to RMW ops or Acquire/Release barriers to keep their semantics. This may be why SeqCst operations are rumored to be "slow" (they really aren't).

# Conclusion

Working with atomic operations requires reasoning about memory very differently than you would normally. You have to take into account both concurrency for the validity of your atomic algorithm, and reordering/visibility for the validity of your algorithm's memory access. It's no wonder that it's considered a challenging topic to tackle.

Hopefully you have Acquired some of this Released information in a way which gives you more visibility into how all of this stuff works. There's more to discuss with atomics than presented here such as how to build correct atomic data structures, handling concurrent memory reclamation, and reducing synchronization. These are all interesting in their own right, but should be saved for another time.

