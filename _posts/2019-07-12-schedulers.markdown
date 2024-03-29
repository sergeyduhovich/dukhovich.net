---
title: "Schedulers"
date: 2019-07-12 12:00:00 +0300
categories: rxswift
toc: true
toc_sticky: true
---

In the last post I covered 2 operators, that work with schedulers. They are `observeOn` and `subscribeOn`. I said that the scheduler might be read as a queue in the context of `observeOn` and `subscribeOn`.

But what are the schedulers? 

If I searched the `SchedulerType` protocol, I would find the `ImmediateSchedulerType` protocol as well. 
If you want to conform the first protocol (write your own scheduler), you have to write 3 methods which do the following:

* dispatch a closure;
* dispatch a closure with a delay;
* dispatch a repetitive closure with a delay;

```swift
func schedule<StateType>(_ state: StateType, action: @escaping (StateType) -> Disposable) -> Disposable
func scheduleRelative<StateType>(_ state: StateType, dueTime: RxTimeInterval, action: @escaping (StateType) -> Disposable) -> Disposable
func schedulePeriodic<StateType>(_ state: StateType, startAfter: RxTimeInterval, period: RxTimeInterval, action: @escaping (StateType) -> StateType) -> Disposable
```

There are some [builtin schedulers](https://github.com/ReactiveX/RxSwift/blob/master/Documentation/Schedulers.md#builtin-schedulers) provided by RxSwift:

### CurrentThreadScheduler

Schedules units of work on the current thread. It's a serial scheduler. Conforms only `ImmediateSchedulerType`, can't be used in repetitive operators.

### SerialDispatchQueueScheduler

The logic here is simple. There is an instance of `DispatchQueue` used under the hood. And for simple closure dispatching `queue.async` is used, and for delay dispatching or repetitive actions `DispatchSourceTimer` is used. 

There is a little optimisation - for the `observeOn` operator, `ObserveOnSerialDispatchQueueSink` and its producer counterpart `ObserveOnSerialDispatchQueue` comes into play.

#### MainScheduler

`MainScheduler` is a subclass of `SerialDispatchQueueScheduler`. `DispatchQueue.main` is used as a backing queue. There is a little optimization - in case schedule methods are called from the main thread, it will perform an action immediately without scheduling. 

### ConcurrentDispatchQueueScheduler

Similarly to `SerialDispatchQueueScheduler`, `queue.async` or `DispatchSourceTimer` is used, but in this case the queue is concurrent.  

### ConcurrentMainScheduler

This scheduler does its work using `MainScheduler` for repetitive actions or dispatches with a delay, or it uses `DispatchQueue.main` for immediate dispatching. This scheduler is optimized for the `subscribeOn` operator.

### OperationQueueScheduler

This scheduler is suitable when you want to limit a number of concurrent threads. It is known, that `DispatchQueue` doesn't provide an API to limit a thread number, but NSOperationQueue does. Can't dispatch work with a delay or repetitive actions, cause it conforms only to `ImmediateSchedulerType`.

### VirtualTimeScheduler

This class conforms to `SchedulerType`, but it doesn't work the same way as other schedulers. It creates an instance of `VirtualSchedulerItem` each time when it needs to dispatch some work. This technique is used for Tests. There is a `TestScheduler` subclass of it in the `RxTest` pod.

### Custom schedulers

You are free to create any custom scheduler that suits your needs. All you need is implementing 3 methods from the `SchedulerType` protocol. As you saw in `VirtualTimeScheduler` implementation, you are not limited to work with real time only. 

## Operators:

There is a list of operators that use schedulers:

* window;
* interval;
* timeout;
* throttle;
* take(duration:);
* subscribeOn; 
* skip(duration:);
* just;
* repeatElement;
* range;
* delay;
* delaySubscription;
* from(optional:);
* debounce;
* buffer.

There is a [handy post](https://www.uraimo.com/2017/05/07/all-about-concurrency-in-swift-1-the-present/) covering almost everything about multithreading in swift.
