---
title: "libuv, but multi-threaded, but not really"
---

What gained popularity as a [LIGMA](https://twitter.com/kingprotty/status/1601991514942488582) 
joke on Twitter morphed into a project I've begun to really consider.

## Background

For context, a few months ago I did a talk at SYCL (software you can love) called [Zig's I/O and Concurrency Story](https://www.youtube.com/watch?v=Ul8OO4vQMTw). This resulted in being approached for advice on how to design high throughput and/or low latency IO systems; Something I'm pretty passionate about at the moment. In the talk, I mention three note-worthy configurations for this problem:

1. Single I/O thread + `N` CPU threads. This should be a default choice.
2. `N` threads sharing I/O and CPU resources. This is your traditional Go, Erlang, and Tokio.
3. `N` threads each with their own I/O resources. This is your Nginx, Seastar, Redpanda and Datadog.

That last one is known as TPC (thread-per-core) and often the most scalable design when it comes to I/O throughput but more importantly tail latency. However, it's not immediately clear why due to lack of work-stealing and the burden of efficient I/O and task distribution being placed on the programmer. I, too,  was skeptical at first until I did some [rudimentary benchmarks](https://gist.github.com/kprotty/5a41e9612657de00788478a7dde43d78) which helped me realize its potential.

But scheduling tasks to properly take advantage of multi-core CPUs in a TPC model isn't always easy or straight-forward. Sometimes, the performance ceiling difference between traditional event loops and TPC for the problem just isn't that much. One could then see how option 2, which maps closely to faimilar OS threads and provides good performance, is so popular and widespread.

## Reasons

In the past, I've dug into various multi-threaded event loops. Each of them have some core inefficiencies that I wish would be corrected. For me, designing scalable systems is more like a form of art than a product enhancement (for hobby code, of course). What it does isn't the fun part. How it achieves what it does is much more interesting. 

This all sounds idealistic, and it kind of is. People writing Rust or Golang don't really care if the underlying runtime is the most well designed system on earth. It simply has to work and be fast. That's fine and all, but it doesn't satisfy my curiousity. I want to see just how far traditional multi-threaded event loops could be pushed to utilize modern computers.

To be honest, I have to admit that some of the drive still stems from trying to feel special. But if you're in Go/Rust/Erlang space, there's not much reasons to use alternative and less popular solutions. I learnt this directly when I designed a [no-`unsafe` async runtime in Rust](https://www.reddit.com/r/rust/comments/riw5oa/tiny_competitive_forbidunsafe_code_async_runtime/). A cool concept, but that was it. Me, being naive, assumed the lack of attention was due to it just not being novel enough. So I did [something similar](https://crates.io/crates/uasync) but this time as a proper crate/package with a smaller scope, less code, and *no dependencies* !!1!!1. In reality, almost no one cares when there's something that already works.

Naturally, I started redirecting this effort to programming spaces that *don't* already have solutions. The most obvious one in my case being Ziglang. Here, I leveraged my experience [writing thread pools](https://zig.news/kprotty/resource-efficient-thread-pools-with-zig-3291), made [dozens](https://gist.github.com/kprotty/08b9bc0658f57cb9412ca48ebe653a66) of [prototypes](https://github.com/kprotty/zap/tree/old_branches) and prompted public discourse on what would constite an ideal runtime ([`#8224](https://github.com/ziglang/zig/issues/8224)). This was even the driver for my talk at SYCL but it unfortunately suffered from lack of direction so nothing *"felt right"*.

## pzero

To actually get anywhere with all these runtime ideas, I eventually had to look backwards. So I reused a dead repo of mine (because project names are hard) to start experimenting. The goal was a runtime like Golang or Tokio but fully intrusive (does no heap allocations), and uses the most efficient path forward when it comes to multi-threaded I/O. Eventually settled on an API like Windows Overlapped/Completion Ports for the latter. 

I've done [a lot](https://gist.github.com/kprotty/08b9bc0658f57cb9412ca48ebe653a66) of [prototyping](https://github.com/kprotty/pzero/branches) on the idea, but unfortunately, interest on my end started to fizzle out again. Decided to just shelf the concept for now until I gain interest later. So instead of a finished library, here's a list of stuff that I learned when designing such a system. These are more like notes to myself so it's fine if it doesn't make any sense:

* Mostly LIFO or mainly FIFO scheduling doesn't seem to matter for the thread-local run queues. If LIFO is used, link the tasks on push so that overflow into injector can be O(1). Also, maintain a `last_target_worker` to avoid rescaning random/empty workers on steal.

* Speaking of work stealing, using `num_workers - 1` for the coprime in the [random array iteration](https://lemire.me/blog/2017/09/18/visiting-all-values-in-an-array-exactly-once-in-random-order/) is a bad idea. Regardless of the random seed, it ends up iterating sequentially in reverse. It really is better to find a coprime from `n/2..n` and cache that.

* Don't overcomplicate the park/unpark primitive exposed to the user. Originally, a lock-free userspace futex impl was planned, but just an event listener is enough. People can write their own sync primitives like mutexes and channels on top of it, even if that won't be the most efficient way to do so.

* Codegen doesn't need to be optimal either. I get this is supposed to be well designed, but stop writing C (and Zig) like it's LLVM IR with heavy type/aliasing annotations, manual overflow subtractions, and branch hints.

* It's not worth optimizing how threads go to sleep. Sure, it would be neat if you could use `NtWaitForAlertByThreadId` and `KUSER_SHARED_DATA.UnparkedProcessorCount` or `ldrex/wfe` on M1 chips to efficiently wait for a condition but 1. no one looks at that code 2. it's ideally a slow path with the atomics guarding it.

* Do the `fetch_sub` notification alg. Hasn't been tested yet, but it's probably better than the CAS based one from [zap](https://github.com/kprotty/zap/). Use CAS for notify() update though; simpler to reason about.

## Ending part

The main take-away for me is that it's fine if this _amazingly designed runtime_ only lives in my mind. The audience isn't there to help guide direction or make it for and the satisfication of solving the hard problem has already been achieved. I think that's enough. Worrying mentally can now be spend on other demands from people like interesting design decisions at TigerBeetle, open source code review when I feel like it, or answering Zig community stuff.

Had to really resist the urge to finish this project and make this blog post a "finished product" like the others. Hopefully this is convincing enough to future me to just write about whatever, whenever I feel like it.

