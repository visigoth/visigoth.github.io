---
layout: post
title: Controlling How NSThread and NSRunLoop Exit
tags:
- ios
- cocoa
- programming
---

While most concurrency can be dealt with by using GCD dispatch queues, there are some things for which you will want to use a separate `NSThread`. For instance, concurrent network programming remains mostly in this area. You may consider using the popular [`CocoaAsyncSocket`](https://github.com/robbiehanson/CocoaAsyncSocket) library, which abstracts away much of the complexity of socket programming on iOS. But, for the sake of this post, assume some work requires a separate thread, and we desire to cleanly start and stop the thread. That is, when we decide to start or stop the thread, we want to ensure it has been initialized or destroyed before moving on.

You can get the final version of the code example below at [github](https://github.com/visigoth/blog-code/tree/master/nsthread_lifecycle).

## Thread Setup

Remember that threads are an artefact of the operating system and not the objective-c runtime. That means all the nice stuff like an `NSAutoreleasePool` and an `NSRunLoop` need to be managed by the thread's top level code. Here is a code snippet of a thread's main method that sets up an autorelease pool as well as starts pumping `NSRunLoop`.

```objc
- (void)start
{
  if (_thread) {
    return;
  }

  _thread = [[NSThread alloc] initWithTarget:self selector:@selector(threadProc:) object:nil];
}

- (void)threadProc:(id)ignored
{
  @autoreleasepool {
    // Startup code here

    // Just spin in a tight loop running the runloop.
    do {
      [[NSRunLoop currentRunLoop] runUntilDate:[NSDate dateWithTimeIntervalSinceNow:1]]
    } while (TRUE);
  }
}
```

This code has many problems. Note that each run of the runloop has a timeout of 1 second. When the runloop has nothing to do for extended periods of time, the thread will still wake up every second and go through the `while` loop's code, only to end up sitting in the runloop's wait condition. That burns CPU and battery. Also, this thread has no exit condition. Even if we want the thread to live until the end of the application's life, we probably want a way to shut it down cleanly. Here's an enumeration of the problems we want to fix:

1. Unsure when the thread is ready for work
1. Never goes to sleep
1. No way to exit the thread cleanly

Solving all three problems at once is tricky.  Fixing #2 means keeping the runloop asleep, preferrably waking only when there is work to be done (being notified by a runloop source). But, that makes it difficult to exit the thread in a timely manner.

## Wait for Infinity

How can we make `NSRunLoop` wait for infinity?  The goal is to make sure that the thread goes to sleep.  Looking at the documentation for `runUntilDate:`, this is not the case:

> If no input sources or timers are attached to the run loop, this method exits immediately; otherwise, it runs the receiver in the NSDefaultRunLoopMode by repeatedly invoking runMode:beforeDate: until the specified expiration date.

The second sentence means this code keeps spinning on the CPU, which is the exact opposite of our desired behavior.  A better alternative is `runMode:beforeDate:`, for which the documentation reads:

> If no input sources or timers are attached to the run loop, this method exits immediately and returns NO; otherwise, it returns after either the first input source is processed or limitDate is reached.

At least there's a chance the thread will sleep when using this method. However, the thread still runs in a continuous loop when there are no sources or timers. If you are only using a run loop to dispatch `performSelector:` calls, you need to add a runloop source of your own to make the thread sleep. As a side effect, it would be effective to use this source to send work to the thread, but that is an exercise left to the reader.

One last thing: what value of `NSDate` should be used? Any large value is fine, as waking the thread once a day is a large enough to keep the thread silent. `+[NSDate distantFuture]` is a convenient factory for such a date.

```objc
static void DoNothingRunLoopCallback(void *info)
{
}

- (void)threadProc:(id)ignored
{
  @autoreleasepool {
    CFRunLoopSourceContext context = {0};
    context.perform = DoNothingRunLoopCallback;

    CFRunLoopSourceRef source = CFRunLoopSourceCreate(NULL, 0, &context);
    CFRunLoopAddSource(CFRunLoopGetCurrent(), source, kCFRunLoopCommonModes);

    do {
      [[NSRunLoop currentRunLoop] runMode:NSDefaultRunLoopMode
                               beforeDate:[NSDate distantFuture]];
    } while (TRUE);

    CFRunLoopRemoveSource(CFRunLoopGetCurrent(), source, kCFRunLoopCommonModes);
    CFRelease(source);
  }
}

```

## The exit condition

Now that the thread sleeps forever, how can we ensure a clean shutdown of the thread? It's entirely possible to use `+[NSThread exit]` to kill the currently running thread. But, this does not clean up stack references to heap objects (like the runloop source that is created), drain the top level autorelease pool, nor allows the thread to flush any remaining work. Instead, we need to make the runloop stop sleeping in order to exit `runMode:beforeDate:`, and we need a condition the thread can check in order to know it's time to shut down.

`NSRunLoop`'s conditions for exiting `runMode:beforeDate:` are somewhat limited. Its return values being `YES` (when the runloop processed an input source or if the timeout was reached) or `NO` (when the runloop could not be started) aren't great. Given that the documentation states that `NO` is returned only when there are no sources or timers, our code will never see `NO` returned since we added an input source to keep the runloop from spinning!

Luckily, `NSRunLoop` controls the same object that `CFRunLoop` APIs do. The CoreFoundation alternative is to use `CFRunLoopRunInMode`, which provides a more specific reason for the runloop's exit. Specifically, `kCFRunLoopRunStopped` indicates the runloop was stopped by `CFRunLoopStop`. This is also the reason that `CFRunLoopRun` exits (other than having no sources or times, but that will not occur due to our fake source), so we don't need to bother with `CFRunLoopRunInMode`, nor check a condition.

It is best to execute `CFRunLoopStop` on the target thread itself. We can do this by using `performSelector:onThread:withObject:waitUntilDone:`.

The new `threadProc:` looks like this:

```objc
- (void)threadProc:(id)ignored
{
  @autoreleasepool {
    CFRunLoopSourceContext context = {0};
    context.perform = DoNothingRunLoopCallback;

    CFRunLoopSourceRef source = CFRunLoopSourceCreate(NULL, 0, &context);
    CFRunLoopAddSource(CFRunLoopGetCurrent(), source, kCFRunLoopCommonModes);

    // Keep processing events until the runloop is stopped.
    CFRunLoopRun();

    CFRunLoopRemoveSource(CFRunLoopGetCurrent(), source, kCFRunLoopCommonModes);
    CFRelease(source);
  }
}
```

We can use code like the following to exit the thread quickly from any thread, including the target thread itself:

```objc
- (void)stop
{
  [self performSelector:@selector(_stop) onThread:_thread withObject:nil waitUntilDone:NO];
  _thread = nil;
}

- (void)_stop
{
  CFRunLoopStop(CFRunLoopGetCurrent());
}
```

## Synchronizing thread startup and exit

Two more problems to solve: when starting the thread, can we ensure that it is ready? When shutting down a thread, can we ensure it is gone?

I would note that there are better threading patterns than attempting to ensure the state of the target thread. For example, a thread should be able to accept work, but simply not process it until it is ready. Resources outside of the thread's runloop should be minimal such that ensuring the thread is no longer executing should be above and beyond the desired knowledge of the thread's state.

It is tempting to simply add `waitUntilDone:YES` to the `performSelector` statement in order to wait until the target thread has exited, but that would only wait until the `_stop` selector was invoked, and not wait for the thread's cleanup code to be run. In order to do that, we need to make a new assumption: the target thread is being shut down from another thread. It would be impossible for a thread to wait for itself to shut down.

In order for the target thread to signal to the control thread that it is done, a condition must be shared between them. `NSCondition` provides convenient semantics for our purposes.

The thread management code is below. This pattern keeps a thread with no work asleep for long periods of time, while allowing for a fast and clean exit. It also allows for startup and shutdown to be synchronized.

```objc
- (void)start
{
  if (_thread) {
    return;
  }

  _thread = [[NSThread alloc] initWithTarget:self selector:@selector(threadProc:) object:nil];

  // _condition was created in -init
  [_condition lock];
  [_thread start];
  [_condition wait];
  [_condition unlock];
}

- (void)stop
{
  if (!_thread) {
    return;
  }

  [_condition lock];
  [self performSelector:@selector(_stop) onThread:_thread withObject:nil waitUntilDone:NO];
  [_condition wait];
  [_condition unlock];
  _thread = nil;
}

- (void)threadProc:(id)object
{
  @autoreleasepool {
    CFRunLoopSourceContext context = {0};
    context.perform = DoNothingRunLoopCallback;

    CFRunLoopSourceRef source = CFRunLoopSourceCreate(NULL, 0, &context);
    CFRunLoopAddSource(CFRunLoopGetCurrent(), source, kCFRunLoopCommonModes);

    [_condition lock];
    [_condition signal];
    [_condition unlock];

    // Keep processing events until the runloop is stopped.
    CFRunLoopRun();

    CFRunLoopRemoveSource(CFRunLoopGetCurrent(), source, kCFRunLoopCommonModes);
    CFRelease(source);

    [_condition lock];
    [_condition signal];
    [_condition unlock];
  }
}
```

## Ensuring Destruction of Thread Resources

There is still another issue with this code: when the thread has signaled its exit, the autorelease pool has not yet been drained. Without ensuring that the thread's memory resources have been released, the purpose of synchronizing the thread's exit becomes much less appealing.

There's a bit of a catch-22. `NSCondition` makes no promise that it is free from using `-autorelease` in its implementations of `-lock`, `-signal`, and `-unlock`. That means there should be a valid `NSAutoreleasePool` when using these APIs. We have two solutions available to us. We can either manually drain the autorelease pool, or use a different way to synchronize the thread's exit that waits until `threadProc:` has exited. The first is somewhat messy. The second has two variants of its own.

### Manual Autorelease

__In order to use `NSAutoreleasePool` directly, you must disable ARC.__

Remember that `-[NSAutoreleasePool drain]` is effectively the same as `-[NSAutoreleasePool release]` and that the pool is no longer valid after draining it. So, manually draining an autorelease pool means creating another one to ensure the `NSCondition` APIs have the right environment.

```objc
- (void)threadProc:(id)object
{
  NSAutoreleasePool *pool = [[NSAutoreleasePool alloc] init];

  {
    CFRunLoopSourceContext context = {0};
    context.perform = DoNothingRunLoopCallback;

    CFRunLoopSourceRef source = CFRunLoopSourceCreate(NULL, 0, &context);
    CFRunLoopAddSource(CFRunLoopGetCurrent(), source, kCFRunLoopCommonModes);

    [_condition lock];
    [_condition signal];
    [_condition unlock];

    // Keep processing events until the runloop is stopped.
    CFRunLoopRun();

    CFRunLoopRemoveSource(CFRunLoopGetCurrent(), source, kCFRunLoopCommonModes);
    CFRelease(source);

    // Release all accumulated resources, but make sure NSCondition has the
    // right environment.
    [pool drain];
    pool = [[NSAutoreleasePool alloc] init];

    [_condition lock];
    [_condition signal];
    [_condition unlock];
  }
  [pool drain];
}
```

### Using NSThreadWillExitNotification

`NSThreadWillExitNotification` is a notification sent by `NSThread` when the thread's main function has finished and the thread is about to finish execution. This must happen after `threadProc:`, so this ensures the thread's top level autorelease pool has been drained. Since the notification fires on the exiting thread, `NSCondition` is still used to synchronize the state of the thread.

```objc
- (void)stop
{
  if (!_thread) {
    return;
  }

  NSNotificationCenter *nc = [NSNotificationCenter defaultCenter];
  [nc addObserver:self selector:@(_signal) name:NSThreadWillExitNotification object:_thread];

  [_condition lock];
  [self performSelector:@selector(_stop) onThread:_thread withObject:nil waitUntilDone:NO];
  [_condition wait];
  [_condition unlock];

  [nc removeObserver:self name:NSThreadWillExitNotification object:_thread];
  _thread = nil;
}

- (void)threadProc:(id)object
{
  @autoreleasepool {
    CFRunLoopSourceContext context = {0};
    context.perform = DoNothingRunLoopCallback;

    CFRunLoopSourceRef source = CFRunLoopSourceCreate(NULL, 0, &context);
    CFRunLoopAddSource(CFRunLoopGetCurrent(), source, kCFRunLoopCommonModes);

    [_condition lock];
    [_condition signal];
    [_condition unlock];

    // Keep processing events until the runloop is stopped.
    CFRunLoopRun();

    CFRunLoopRemoveSource(CFRunLoopGetCurrent(), source, kCFRunLoopCommonModes);
    CFRelease(source);
  }
}

- (void)_signal
{
  [_condition lock];
  [_condition signal];
  [_condition unlock];
}
```

### Using pthreads

All of the above solutions have one slight problem: the thread hasn't quite exited when the control thread believes it has. The target thread is very close to being done, but that is not the same as done.

To ensure that, we must go outside the realm of `NSThread` and use pthreads instead. `pthread_join` guarantees the target thread has terminated. Using pthreads makes the code a bit more verbose, and some of the memory management must be done carefully. In particular, `self` is retained when specifying it as an argument of `NSThread`'s initializer, but will not be retained when using it as an argument to `pthread_create`. Note that we still need an `NSThread` reference to use `performSelector:onThread:withObject:waitUntilDone:`, but there is no way to convert a `pthread_t` to an `NSThread`. Luckily, `+[NSThread currentThread]` can obtain the correct object reference.

`NSCondition` is still used to synchronize thread startup. Because it is not used for any other purpose, it is not necessary to lock the condition before starting the thread. However, to be consistent with previous code, we will follow the previous pattern of creating the thread in a suspended state and resuming it with the condition's lock held.

```objc
static void *ThreadProc(void *arg)
{
  ThreadedComponent *component = (__bridge_transfer ThreadedComponent *)arg;
  [component threadProc:nil];
  return 0;
}

- (void)start
{
  if (_thread) {
    return;
  }

  if (pthread_create_suspended_np(&_pthread, NULL, &ThreadProc, (__bridge_retained void *)self) != 0) {
    return;
  }

  // _condition was created in -init
  [_condition lock];
  mach_port_t mach_thread = pthread_mach_thread_np(_pthread);
  thread_resume(mach_thread);
  [_condition wait];
  [_condition unlock];
}

- (void)stop
{
  if (!_thread) {
    return;
  }

  [self performSelector:@selector(_stop) onThread:_thread withObject:nil waitUntilDone:NO];
  pthread_join(_pthread, NULL);
  _thread = nil;
}

- (void)threadProc:(id)object
{
  @autoreleasepool {
    CFRunLoopSourceContext context = {0};
    context.perform = DoNothingRunLoopCallback;

    CFRunLoopSourceRef source = CFRunLoopSourceCreate(NULL, 0, &context);
    CFRunLoopAddSource(CFRunLoopGetCurrent(), source, kCFRunLoopCommonModes);

    // Obtain the current NSThread before signaling startup is complete.
    _thread = [NSThread currentThread];

    [_condition lock];
    [_condition signal];
    [_condition unlock];

    // Keep processing events until the runloop is stopped.
    CFRunLoopRun();

    CFRunLoopRemoveSource(CFRunLoopGetCurrent(), source, kCFRunLoopCommonModes);
    CFRelease(source);
  }
}

```

## Code Samples

Head on over to [github](https://github.com/visigoth/blog-code/tree/master/nsthread_lifecycle) to download code samples for all four working versions of the code above.
