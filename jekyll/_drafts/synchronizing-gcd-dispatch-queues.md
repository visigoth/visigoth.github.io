---
layout: post
title: Synchronizing GCD Dispatch Queues
tags:
- ios
- cocoa
- gcd
---

Let's consider some use cases of GCD dispatch queues. They are often used in producer-consumer relationships for processing data or doing asynchronous work, either in a serial or parallel fashion.

In rare cases, you may need to synchronize the work on your queue with another thread. However, a dispatch queue does not operate on any given thread, and combining synchronization primitives with each block placed on the queue would be complicated and fragile.

A better method is to pause and resume the execution of a dispatch queue using `dispatch_suspend` and `dispatch_resume`.

## `dispatch_suspend` is Asynchronous

But, there's something curious about `dispatch_suspend`. Apple's documentation for the function says:

> The suspension occurs after completion of any blocks running at the time of the call.

The documentation doesn't tell us whether `dispatch_suspend` waits for these blocks to complete before returning, so I created a small [test program](https://github.com/visigoth/blog-code/blob/master/dispatch_suspend/main.m) to explore this. It tests this by queueing a block that waits on a condition that only becomes true if `dispatch_suspend` is asynchronous. If `dispatch_suspend` is synchronous, the application will hang.

When running this program on OS X 10.9, the program shows `dispatch_suspend` is an asynchronous process, so it can't be used on its own to synchronize a thread with work on a dispatch queue directly. However, there is a way we can build on top of it.

## Barrier Blocks

Another little-known facility provided by dispatch queues are barrier blocks. Barrier blocks work on both serial and concurrent dispatch queues. When submitted to a queue, all blocks ahead of the barrier will complete before the barrier starts execution, and all blocks behind the barrier will only start executing once the barrier is complete. For serial queues, this is the same as queueing any block. For concurrent queues, this is especially handy for our purposes. Submitting a barrier block can be done with `dispatch_barrier_async` and `dispatch_barrier_sync` (and their function variants).

## Synchronizing Queues

Armed with this knowledge, we can synchronize the work done on a dispatch queue with an external thread like so:

{% highlight objc %}
dispatch_barrier_sync(queue, ^{ dispatch_suspend(queue); });

// Do some stuff

dispatch_resume(queue);
{% endhighlight %}
