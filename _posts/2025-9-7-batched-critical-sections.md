---
title: "Batched Critical Sections"
---

Critical sections don't have to be scheduler-bottlenecked.

### What do you mean?

First off, a critical section is just a piece of code that multiple threads want to run, but only one thread at a time is allowed to execute. The most obvious way to achieve this is through a Mutex, but it's not really ideal:

The flow of protected code through a Mutex looks like this:
```c
update(t1) // locked
wake(t2): // unlock
    update(t2)
    wake(t3):
        update(t3)
```
Each indentation level represents a different thread, where it must wait to be scheduled after its parent's `wake`. This sucks because the critical sections we care about (the `update` calls) are separated by points that wait for the scheduler to start running a woken up thread.

What if, instead, you run all the updates together on one thread, then do the `wake`s after? It would remove the dependency on the scheduler's decision to run us entirely:
```c
update(t1)
update(t2)
update(t3)
can_be_in_parallel { wake(t2), wake(t3) }
```

I've started calling this pattern **Batched Critical Sections** (BCS).

### How would you do that?
People already do this. Well.. something close it with [Actors](https://en.wikipedia.org/wiki/Actor_model). 

Under this model, there's a Multi-Producer Single-Consumer (MPSC) queue where many threads can push data/operations (messages) while a single consumer thread continuously pops and handles them:
```c
can_be_in_parallel { push(t1), push(t2), push(t2) }
wake(consumer):
    update(t1)
    update(t2)
    update(t3)
    can_be_in_parallel { wake(t1), wake(t2), wake(t3) }
```

It's the right start, but there's still that scheduler dependency on waking the consumer thread to pop. Turns out you can just skip that; Instead of another consumer thread, the first thread to push when the queue is empty **becomes the consumer**, popping & handling messages as the critical section until the queue is empty. So as pseudo code:
```c
submit(t1):
  was_empty = mpsc.push(t1)
  if was_empty:
    while mpsc.pop() |t|:
      update(t)
      // defer wake(t) after loop
```

But this isn't enough. A careful reader might've notice that there's a bug here where multiple producers could enter the consumer path:
```
t1: push(t1), was_empty=True, pops t1, is preempted
t2: push(t2), was_empty=True, pops t2
  t1 runs update(t1) & t2 runs update(t2) concurrently to each other
```

### Is there a fix?

Yea. The consumer needs support a two-phase `pop()` process, where it first observes a message then marks it as popped after handling it. Rather than showing another abstract version of that, let's just get into a concrete implementation:

<details>
<summary>Intrusive, Lock-Free, BCS </summary>

```rs
/// Glossary:
/// ?T = T or null
/// cmpxchg() returns ?T: success=null fail=T (new value)

struct Node:
  next: ?*Node

struct Queue:
  top: Atomic(?*Node)

  submit(self, node: *Node):
    // Classic lock-free push
    top = self.top.load()
    loop:
      node.next = top
      top = self.top.cmpxchg(top, node) orelse break

    // If was_empty
    if top == null:
      consumer(node, null)

  consumer(self, top: *Node, bottom: ?*Node):
    // Handle all nodes that were pushed so far.
    node = top
    do:
      next = node.next
      handle(node) // defer node.wake() after our return
      invalidate(node)
      node = next
    while node != bottom
    
    // Try to mark nodes as popped. Retries if any new nodes are pushed.
    new_top = self.top.cmpxchg(top, null) orelse return
    consumer(new_top, top)
```

</details>

<hr />

That's it. 

That's the whole alg. It has some nice properties too:

* The critical section (`handle`) is allowed to invalidate the nodes. So a decision to immediately `wake`, or defer the `wake`, can be made per-node.
* Critical section state can be threaded through `Queue.top` by representing the "empty" state, as long as it fits in a pointer and is distinguishable from submitted node pointers.

I want to further emphasize that this is more than just a _"better Mutex"_ algorithm; **It's a concurrency pattern**. Particularly useful for protecting _other_ intrusive data structures like queues or graphs.

For example, submit a waiter to a wait-queue, a callback node to an `io_uring` instance, or a timer node to a tree. Basically, anywhere you'd use a Mutex or an MPSC channel. It's asynchronous by default and can optionally be made synchronous by waiting on what's submitted (waking it when it's handled by the critical section).

> **Sidenote**: 
> It's wild that the OS with the best thread park/unpark API for this is NetBSD of all things. Batched Critical Sections use the [Event](https://kprotty.me/2025/07/31/sync-primitives-are-functionally-complete.html) pattern i've described prior. And only Windows & NetBSD support that natively, with the latter having [`lwp_unpark_all()`](https://man.netbsd.org/_lwp_unpark_all.2) for batched `wake()`. Everyone else uses [`futex`](https://en.wikipedia.org/wiki/Futex) as the lowest-level park/unpark primitive.

### Can it be faster?

With some trade-offs? Sort of.

The main area for improvement is that `submit()` is currently lock-free when it could be wait-free. BCS at it's core is just an intrusive lock-free MPSC. So let's instead take inspiration from another intrusive _wait-free_ MPSC:

The [Vyukov Queue](https://int08h.com/post/ode-to-a-vyukov-queue/) is just that, distilled to only the necessary operations. Here's it adapted into a more traditional API:

<details>
<summary>Intrusive, Wait-Free, MPSC</summary>

```rs
struct Node:
  next: Atomic(?*Node)

struct Queue:
  head: Atomic(?*Node)
  tail: Atomic(?*Node)

  push(self, node: *Node):
    // Push node to tail
    node.next = null
    if self.tail.swap(node) |prev_tail|:
      // non-empty: link to previous tail
      prev_tail.next.store(node)
    else:
      // empty: set a head
      self.head.store(node)

  pop(self) -> !*Node:
    node = self.head.load() orelse return Empty
    self.head = node.next.load() orelse blk:
      // no next, should be the last
      self.head = null
      _ = self.tail.cmpxchg(node, null) orelse return node
      // new push. it should set node.next
      self.head = node
      break :blk node.next.load() orelse return ProducerStalled
    return node
```

</details>

A quirk of this queue is that it introduces a new failure mode for the consumer: `ProducerStalled`. This happens when the consumer is on some/last node, a producer pushes a new one to tail with `swap`, but hasn't yet linked it to the previous tail with `store`. 

Given the window for this to happen is so small (like a few instructions), most Vyukov queue implementations spin on the that `prev_tail.next` until the producer sets it, relying on OS preemption or parallelism to eventually make it visible. While that probably works in practice, unbounded spinning technically downgrades the forward progress guarantess from "_Wait Free_" to "_Blocking_" which is cringe.

But when adapted to BCS, we can work around it; When the consumer gets into that case, it will atomically swap `prev_tail.next` with some sentinel. The producer side also now links to previous tail with a `swap`. If the consumer sees the producer's swap, it continues with the next node as normal. If not, the consumer returns and knows the producer will see sentinel to continue where it left off as the _new consumer_.

<details>
<summary>Intrusive, Wait-Free, BCS</summary>

```rs
struct Node:
  next: Atomic(?*Node)

struct Queue:
  tail: Atomic(?*Node)

  submit(self, node: *Node):
    // Push node to tail, if empty become consumer
    node.next = null
    prev = self.tail.swap(node) orelse:
        return consumer(node)

    // Link to prev tail. If set, continue handoff from prev consumer.
    if prev.next.swap(node) == Sentinel:
        invalidate(prev)
        return consumer(node)

  consumer(self, node: ?*Node):
    handle(node)
    next = node.next.load() orelse blk:
        // looks like last node, try to remove it from queue.
        _ = self.tail.cmpxchg(node, null) orelse return invalidate(node) // done
        // new push happened. either observe it & continue or handoff consumer
        break :blk node.next.swap(Sentinel) orelse return // handoff
    invalidate(node)
    consumer(next)
```

</details>

The tradeoff is that the critical section (`handle`) can no longer invalidate the nodes. But this is usually fine as the invalidation points are still explicit and happen during the consumer's ownership period (exclusing the last node). 

If you really wanted that property however, it can still be done; Observe `node.next` first & if null do the `tail.cmpxchg` but to a stub node instead of null. On success, `handle(last_node)` as usual, then `tail.cmpxchg(stub, null)` to complete. Failing this does the handoff as usual, but the new consumer skips handling the `stub`. The stub node itself must live longer than the last dequeue so itll probably need to live in the Queue itself. But with this, `handle` should now support node invalidation.

### Ok. Any good in practice?

Yes.

I implemented a mutex-like critical section with both BCS variants in Rust and benchmarked it against other mutex-based ones like `std::sync::Mutex`, `parking_lot::Mutex`, and `pthread_mutex_t/SRWLOCK`. 

The hypothesis was that it should have around the same throughput as [unfair locks](https://www.intel.com/content/www/us/en/docs/onetbb/developer-guide-api-reference/2021-6/mutex-flavors.html) but with better fairness from queued critical sections being processed in FIFO order instead of scheduler-preference order. 

[Async Mutex use]

### I want to use it.

[Library, Code]