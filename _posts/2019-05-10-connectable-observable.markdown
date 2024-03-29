---
title: "Connectable Observable"
date: 2019-05-10 12:00:00 +0300
categories: rxswift
toc: true
toc_sticky: true
---

There are only 4 operators in the [connectable](http://reactivex.io/documentation/operators.html#connectable) category.

In the [introduction post]({% post_url 2019-04-26-rxswift-introduction %}) I mentioned, that there are 2 Observable types: cold and hot. By default all [creating operators]({% post_url 2019-05-03-creating-an-observable %}) return a cold Observable. What does it mean?

Let's take a look at the following example. I created a timer to do some work every second:

```swift
Observable<Int>.interval(1, scheduler: ConcurrentDispatchQueueScheduler(qos: .background)) // creation
  .debug("interval 1")
  .subscribe() // subscription
  .disposed(by: disposeBag)
```

Observable was created in line 1, and because we have `subscribe` call, it started emitting `next` events. At this moment everything works as expected and in the console we see:

```swift
// interval 1 -> subscribed
// interval 1 -> Event next(0)
// interval 1 -> Event next(1)
// interval 1 -> Event next(2)
// interval 1 -> Event next(3)
// interval 1 -> Event next(4)
```

But if we need more than one Observable to subscribe (probably after some transformation) and we don't want to create another timer for it, we update the code above to something like this:

```swift
let tickObservable: Observable<Int> = Observable<Int>.interval(1, scheduler: ConcurrentDispatchQueueScheduler(qos: .background))
    .debug("interval 1")

let strObservable: Observable<String> = tickObservable
  .map { "currently we at \($0) tick" }

tickObservable
  .debug("tickObservable")
  .subscribe()
  .disposed(by: disposeBag)

strObservable
  .debug("strObservable")
  .subscribe()
  .disposed(by: disposeBag)
```

After the creation I assigned Observable to the `tickObservable` variable and used it to create another one using `map` operator. No big deal. At first glance it looks meaningful. Right now I have 2 Observables and I want to call `subscribe` on both of them. Ok, let's check out the console:

```swift
//1
// tickObservable -> subscribed
// interval 1 -> subscribed
//2
// strObservable -> subscribed
// interval 1 -> subscribed
//3
// interval 1 -> Event next(0)
//4
// tickObservable -> Event next(0)
//5
// interval 1 -> Event next(0)
//6
// strObservable -> Event next(currently we at 0 tick)
//
// interval 1 -> Event next(1)
// interval 1 -> Event next(1)
// strObservable -> Event next(currently we at 1 tick)
// tickObservable -> Event next(1)
// interval 1 -> Event next(2)
// strObservable -> Event next(currently we at 2 tick)
// interval 1 -> Event next(2)
// tickObservable -> Event next(2)
```

### Let's go through the first six comments.

For each `tickObservable` and `strObservable` we can see `subscribed` debug print. And as each of them is the result of 2 chained `debug` operators we see `interval 1 -> subscribed` as well.

```swift
tickObservable
  .debug("tickObservable")
  .subscribe()
  .disposed(by: disposeBag)
  
//is the same as
Observable<Int>.interval(1, scheduler: ConcurrentDispatchQueueScheduler(qos: .background))
  .debug("interval 1")
  .debug("tickObservable")
  .subscribe()
  .disposed(by: disposeBag)
  
//and
strObservable
  .debug("strObservable")
  .subscribe()
  .disposed(by: disposeBag)
  
//is the same as
Observable<Int>.interval(1, scheduler: ConcurrentDispatchQueueScheduler(qos: .background))
  .debug("interval 1")
  .map { "currently we at \($0) tick" }
  .debug("strObservable")
  .subscribe()
  .disposed(by: disposeBag)
```

And because of the chained `debug` operator, we see 2 prints for each event. The third  and the fourth comments represent the first `next` event for the chained `tickObservable`. The fifth and the sixth comments represent the first `next` event for the chained `strObservable`.

### The problem

Wait a second. In the previous paragraph I said (and the console log approved) that there are 2 prints for each `tickObservable` and `strObservable` or 4 prints in total. It is not exactly what I wanted. I wanted one print for `interval 1`, one for `tickObservable` and one for `strObservable`, 3 prints in total. 4 prints mean that we have not one, but two timers at the same time. Each `subscribe` call triggers the creation operator `interval(1, scheduler: ConcurrentDispatchQueueScheduler(qos: .background))`, as a result in our case it created 2 timers instead of one.

### The solution

In the [documentation](https://github.com/ReactiveX/RxSwift/blob/master/Documentation/GettingStarted.md#sharing-subscription-and-share-operator) we can find `Sharing subscription and share operator` section. The easiest way to fix our problem is using `share()` operator. The following example is absolutely the same but there is a small modification on `tickObservable` variable - we added `share()` operator at the end.

```swift
let tickObservable: Observable<Int> = Observable<Int>.interval(1, scheduler: ConcurrentDispatchQueueScheduler(qos: .background))
    .debug("interval 1")
    .share() // added share() operator

let strObservable: Observable<String> = tickObservable
  .map { "currently we at \($0) tick" }

tickObservable
  .debug("tickObservable")
  .subscribe()
  .disposed(by: disposeBag)

strObservable
  .debug("strObservable")
  .subscribe()
  .disposed(by: disposeBag)
```

Let's check out the console:

```swift
// tickObservable -> subscribed
// interval 1 -> subscribed
// strObservable -> subscribed
//
// interval 1 -> Event next(0)
// tickObservable -> Event next(0)
// strObservable -> Event next(currently we at 0 tick)
//
// interval 1 -> Event next(1)
// tickObservable -> Event next(1)
// strObservable -> Event next(currently we at 1 tick)
//
// interval 1 -> Event next(2)
// tickObservable -> Event next(2)
// strObservable -> Event next(currently we at 2 tick)
```

There are exactly 3 prints for each event, one for the shared timer, one for `tickObservable` and one for `strObservable`. Looks like our problem is fixed.

### Connectable Operators

It's time to check out available [connectable operators](http://reactivex.io/documentation/operators.html#connectable). They are:

* Connect — instructs a connectable Observable to begin emitting items to its subscribers
* Publish — converts an ordinary Observable into a connectable Observable
* RefCount — makes a Connectable Observable behave like an ordinary Observable
* Replay — ensures that all observers see the same sequence of emitted items, even if they subscribe after the Observable has begun emitting items

Hm, there is nothing about `share` operator. What is it? We can find the method in the `ShareReplayScope.swift` file.

```swift
public func share(replay: Int = 0, scope: SubjectLifetimeScope = .whileConnected) -> Observable<E> {
  switch scope {
  case .forever:
    switch replay {
    case 0: return self.multicast(PublishSubject()).refCount()
    default: return self.multicast(ReplaySubject.create(bufferSize: replay)).refCount()
    }
  case .whileConnected:
    switch replay {
    case 0: return ShareWhileConnected(source: self.asObservable())
    case 1: return ShareReplay1WhileConnected(source: self.asObservable())
    default: return self.multicast(makeSubject: { ReplaySubject.create(bufferSize: replay) }).refCount()
    }
  }
}
```

Ok, `share` is a wrapper for the combination of `multicast` + `refCount`. There are 2 optimized versions for `share(replay: 1)` and `share()`, others are the combinations of the calls `multicast`.`refCount`.

### Multicast / Publish

Let's open the `Multicast.swift` file.

```swift
public func publish() -> ConnectableObservable<E> {
  return self.multicast { PublishSubject() }
}
```

Publish is a multicast using `PublishSubject`. Publish and multicast convert Observable to Connectable Observable, which is:

```swift
public protocol ConnectableObservableType : ObservableType {
  func connect() -> Disposable
}
```

Connectable Observable doesn't begin emitting events immediately, it waits for instructions which can be `connect` method or `refCount`.

### Observable lifetime

There are two different approaches to manage Observable lifetime in RxSwift.

### Connect 

#### Scenario 1, S2 subscribes before S1 unsubscribes:

![connect](http://dukhovich.by/assets/images/articles/connect_1.jpg)

```swift
    let connectable = Observable<Int>.interval(1, scheduler: ConcurrentDispatchQueueScheduler(qos: .background))
      .debug("interval 1")
      .multicast(PublishSubject<Int>())

    _ = connectable.connect()

    DispatchQueue.main
      .asyncAfter(deadline: DispatchTime.now() + 1.5) {
        connectable
          .debug("S1")
          .subscribe()
          .disposed(by: self.disposeBagS1)
    }

    DispatchQueue.main
      .asyncAfter(deadline: DispatchTime.now() + 3.5) {
        self.disposeBagS1 = DisposeBag()
    }

    DispatchQueue.main
      .asyncAfter(deadline: DispatchTime.now() + 2.5) {
        connectable
          .debug("S2")
          .subscribe()
          .disposed(by: self.disposeBagS2)
    }

    DispatchQueue.main
      .asyncAfter(deadline: DispatchTime.now() + 6.5) {
        self.disposeBagS2 = DisposeBag()
    }
```

it prints:

```swift
//interval 1 -> subscribed
//interval 1 -> Event next(0)
//S1 -> subscribed
//interval 1 -> Event next(1)
//S1 -> Event next(1)
//S2 -> subscribed
//interval 1 -> Event next(2)
//S1 -> Event next(2)
//S2 -> Event next(2)
//S1 -> isDisposed
//interval 1 -> Event next(3)
//S2 -> Event next(3)
//interval 1 -> Event next(4)
//S2 -> Event next(4)
//interval 1 -> Event next(5)
//S2 -> Event next(5)
//S2 -> isDisposed
//interval 1 -> Event next(6)
//interval 1 -> Event next(7)
//interval 1 -> Event next(8)
//interval 1 -> Event next(9)
```

#### Scenario 2, S2 subscribes after S1 unsubscribes:

![connect](http://dukhovich.by/assets/images/articles/connect_2.jpg)

```swift
    let connectable = Observable<Int>.interval(1, scheduler: ConcurrentDispatchQueueScheduler(qos: .background))
      .debug("interval 1")
      .multicast(PublishSubject<Int>())

    _ = connectable.connect()

    DispatchQueue.main
      .asyncAfter(deadline: DispatchTime.now() + 1.5) {
        connectable
          .debug("S1")
          .subscribe()
          .disposed(by: self.disposeBagS1)
    }

    DispatchQueue.main
      .asyncAfter(deadline: DispatchTime.now() + 3.5) {
        self.disposeBagS1 = DisposeBag()
    }

    DispatchQueue.main
      .asyncAfter(deadline: DispatchTime.now() + 4.5) {
        connectable
          .debug("S2")
          .subscribe()
          .disposed(by: self.disposeBagS2)
    }

    DispatchQueue.main
      .asyncAfter(deadline: DispatchTime.now() + 6.5) {
        self.disposeBagS2 = DisposeBag()
    }
```

it prints:

```swift
//interval 1 -> subscribed
//interval 1 -> Event next(0)
//S1 -> subscribed
//interval 1 -> Event next(1)
//S1 -> Event next(1)
//interval 1 -> Event next(2)
//S1 -> Event next(2)
//S1 -> isDisposed
//interval 1 -> Event next(3)
//S2 -> subscribed
//interval 1 -> Event next(4)
//S2 -> Event next(4)
//interval 1 -> Event next(5)
//S2 -> Event next(5)
//S2 -> isDisposed
//interval 1 -> Event next(6)
//interval 1 -> Event next(7)
//interval 1 -> Event next(8)
//interval 1 -> Event next(9)
```

OK. How could we describe `connect` operator?

* Connect instructs a connectable Observable to begin emitting events to its subscribers;
* Connectable Observable became a real hot Observable, which means it doesn't wait for `subscribe` call;
* Doesn't stop producing events when the number of subscribers drops to zero;
* Doesn't recreate an underlying Observable when the number of subscribers increases from 0 to 1;

### RefCount

Let's take a look at `refCount`:

#### Scenario 1, S2 subscribes before S1 unsubscribes:

![refCount](http://dukhovich.by/assets/images/articles/ref_count_2.jpg)

```swift
    let connectable = Observable<Int>.interval(1, scheduler: ConcurrentDispatchQueueScheduler(qos: .background))
      .debug("interval 1")
      .multicast(PublishSubject<Int>())

    let refCount = connectable.refCount()

    DispatchQueue.main
      .asyncAfter(deadline: DispatchTime.now() + 1.5) {
        refCount
          .debug("S1")
          .subscribe()
          .disposed(by: self.disposeBagS1)
    }

    DispatchQueue.main
      .asyncAfter(deadline: DispatchTime.now() + 4) {
        self.disposeBagS1 = DisposeBag()
    }

    DispatchQueue.main
      .asyncAfter(deadline: DispatchTime.now() + 3) {
        refCount
          .debug("S2")
          .subscribe()
          .disposed(by: self.disposeBagS2)
    }

    DispatchQueue.main
      .asyncAfter(deadline: DispatchTime.now() + 7) {
        self.disposeBagS2 = DisposeBag()
    }
```

it prints:

```swift
//S1 -> subscribed
//interval 1 -> subscribed
//interval 1 -> Event next(0)
//S1 -> Event next(0)
//S2 -> subscribed
//interval 1 -> Event next(1)
//S1 -> Event next(1)
//S2 -> Event next(1)
//S1 -> isDisposed
//interval 1 -> Event next(2)
//S2 -> Event next(2)
//interval 1 -> Event next(3)
//S2 -> Event next(3)
//interval 1 -> Event next(4)
//S2 -> Event next(4)
//S2 -> isDisposed
//interval 1 -> isDisposed
```

#### Scenario 2, S2 subscribes after S1 unsubscribes:

![refCount](http://dukhovich.by/assets/images/articles/ref_count_1.jpg)

```swift
    let connectable = Observable<Int>.interval(1, scheduler: ConcurrentDispatchQueueScheduler(qos: .background))
      .debug("interval 1")
      .multicast(PublishSubject<Int>())

    let refCount = connectable.refCount()

    DispatchQueue.main
      .asyncAfter(deadline: DispatchTime.now() + 1.5) {
        refCount
          .debug("S1")
          .subscribe()
          .disposed(by: self.disposeBagS1)
    }

    DispatchQueue.main
      .asyncAfter(deadline: DispatchTime.now() + 4) {
        self.disposeBagS1 = DisposeBag()
    }

    DispatchQueue.main
      .asyncAfter(deadline: DispatchTime.now() + 5.5) {
        refCount
          .debug("S2")
          .subscribe()
          .disposed(by: self.disposeBagS2)
    }

    DispatchQueue.main
      .asyncAfter(deadline: DispatchTime.now() + 8) {
        self.disposeBagS2 = DisposeBag()
    }
```

it prints:

```swift
//S1 -> subscribed
//interval 1 -> subscribed
//interval 1 -> Event next(0)
//S1 -> Event next(0)
//interval 1 -> Event next(1)
//S1 -> Event next(1)
//S1 -> isDisposed
//interval 1 -> isDisposed
//S2 -> subscribed
//interval 1 -> subscribed
//interval 1 -> Event next(0)
//S2 -> Event next(0)
//interval 1 -> Event next(1)
//S2 -> Event next(1)
//S2 -> isDisposed
//interval 1 -> isDisposed
```

OK. How could we describe `refCount` operator?

* Begins emitting events when the first subscriber calls `subscribe`
* Stops producing events when the number of subscribers drops to zero;
* Recreates an underlying Observable when the number of subscribers increases from 0 to 1;

### Replay

We've covered 3 of 4 operators from the connectable section.

In the examples above we have used PublishSubject. To see how `replay` works, we should pass `ReplaySubject` instance as a parameter to `multicast` function. Replay works a bit differently with `connect` and `refCount`. Let's check the difference.

#### Case 1

We call subscribe as the first subscriber after 2.5 seconds to `connect` and `refCount`, both have `ReplaySubject` with 2 last items:

![replay](http://dukhovich.by/assets/images/articles/replay_1.jpg)

`RefCount` waits for the first subscriber and by the time it subscribes there is nothing to replay:

```swift
    let connectable = Observable<Int>.interval(1, scheduler: ConcurrentDispatchQueueScheduler(qos: .background))
      .debug("interval 1")
      .multicast(ReplaySubject<Int>.create(bufferSize: 2))

    let refCount = connectable.refCount()

    DispatchQueue.main
      .asyncAfter(deadline: DispatchTime.now() + 2.5) {
        refCount
          .debug("S1")
          .subscribe()
          .disposed(by: self.disposeBagS1)
    }
```

```swift
//S1 -> subscribed
//interval 1 -> subscribed
//interval 1 -> Event next(0)
//S1 -> Event next(0)
//interval 1 -> Event next(1)
//S1 -> Event next(1)
//interval 1 -> Event next(2)
//S1 -> Event next(2)
//interval 1 -> Event next(3)
//S1 -> Event next(3)
```

`Connect` doesn't wait for subscribers. It starts emitting events. And because we gave some extra odds for the first subscriber there are 2 items replayed immediately after the subscription:

```swift
    let connectable = Observable<Int>.interval(1, scheduler: ConcurrentDispatchQueueScheduler(qos: .background))
      .debug("interval 1")
      .multicast(ReplaySubject<Int>.create(bufferSize: 2))

    _ = connectable.connect()

    DispatchQueue.main
      .asyncAfter(deadline: DispatchTime.now() + 2.5) {
        connectable
          .debug("S1")
          .subscribe()
          .disposed(by: self.disposeBagS1)
    }
```

```swift
//interval 1 -> subscribed
//interval 1 -> Event next(0)
//interval 1 -> Event next(1)
//S1 -> subscribed
//S1 -> Event next(0)
//S1 -> Event next(1)
//interval 1 -> Event next(2)
//S1 -> Event next(2)
//interval 1 -> Event next(3)
//S1 -> Event next(3)
//interval 1 -> Event next(4)
//S1 -> Event next(4)
//interval 1 -> Event next(5)
//S1 -> Event next(5)
//interval 1 -> Event next(6)
//S1 -> Event next(6)
```

#### Case 2

We call subscribe as the second subscriber after 3.5 seconds to `connect` and `refCount`, both have `ReplaySubject` with 2 last items:

![replay](http://dukhovich.by/assets/images/articles/replay_2.jpg)

```swift
let connectable = Observable<Int>.interval(1, scheduler: ConcurrentDispatchQueueScheduler(qos: .background))
      .debug("interval 1")
      .multicast(ReplaySubject<Int>.create(bufferSize: 2))

    _ = connectable.connect()

    connectable
      .debug("S1")
      .subscribe()
      .disposed(by: self.disposeBagS1)

    DispatchQueue.main
      .asyncAfter(deadline: DispatchTime.now() + 3.5) {
        connectable
          .debug("S2")
          .subscribe()
          .disposed(by: self.disposeBagS2)
    }
```

```swift
    let connectable = Observable<Int>.interval(1, scheduler: ConcurrentDispatchQueueScheduler(qos: .background))
      .debug("interval 1")
      .multicast(ReplaySubject<Int>.create(bufferSize: 2))

    let refCount = connectable.refCount()

    refCount
      .debug("S1")
      .subscribe()
      .disposed(by: self.disposeBagS1)

    DispatchQueue.main
      .asyncAfter(deadline: DispatchTime.now() + 3.5) {
        refCount
          .debug("S2")
          .subscribe()
          .disposed(by: self.disposeBagS2)
    }
```

There is no difference in the console. By the time the second subscriber calls `subscribe` there are 2 items to replay for both: `connect` and `refCount`:

```swift
//interval 1 -> subscribed
//S1 -> subscribed
//interval 1 -> Event next(0)
//S1 -> Event next(0)
//interval 1 -> Event next(1)
//S1 -> Event next(1)
//interval 1 -> Event next(2)
//S1 -> Event next(2)
//S2 -> subscribed
//S2 -> Event next(1)
//S2 -> Event next(2)
//interval 1 -> Event next(3)
//S1 -> Event next(3)
//S2 -> Event next(3)
//interval 1 -> Event next(4)
//S1 -> Event next(4)
//S2 -> Event next(4)
//interval 1 -> Event next(5)
//S1 -> Event next(5)
//S2 -> Event next(5)
//interval 1 -> Event next(6)
//S1 -> Event next(6)
//S2 -> Event next(6)
```

#### Case 3

We call subscribe as the second subscriber after 14.5 seconds while the first subscriber was unsubscribed right after receiving `next(10)`:

![replay](http://dukhovich.by/assets/images/articles/replay_3.jpg)

```swift
    let connectable = Observable<Int>.interval(1, scheduler: ConcurrentDispatchQueueScheduler(qos: .background))
      .debug("interval 1")
      .multicast(ReplaySubject<Int>.create(bufferSize: 2))

    _ = connectable.connect()

    connectable
      .debug("S1")
      .subscribe()
      .disposed(by: self.disposeBagS1)

    DispatchQueue.main
      .asyncAfter(deadline: DispatchTime.now() + 11.5) {
        self.disposeBagS1 = DisposeBag()
    }

    DispatchQueue.main
      .asyncAfter(deadline: DispatchTime.now() + 14.5) {
        connectable
          .debug("S2")
          .subscribe()
          .disposed(by: self.disposeBagS2)
    }
```

There is nothing worth discussing for `connect` operator. This scenario is very much the same as the first and second ones:

```swift
//interval 1 -> Event next(7)
//S1 -> Event next(7)
//interval 1 -> Event next(8)
//S1 -> Event next(8)
//interval 1 -> Event next(9)
//S1 -> Event next(9)
//interval 1 -> Event next(10)
//S1 -> Event next(10)
//S1 -> isDisposed
//interval 1 -> Event next(11)
//interval 1 -> Event next(12)
//interval 1 -> Event next(13)
//interval 1 -> Event next(14)
//S2 -> subscribed
//S2 -> Event next(13)
//S2 -> Event next(14)
//interval 1 -> Event next(15)
//S2 -> Event next(15)
//interval 1 -> Event next(16)
//S2 -> Event next(16)
//interval 1 -> Event next(17)
//S2 -> Event next(17)
//interval 1 -> Event next(18)
//S2 -> Event next(18)
//interval 1 -> Event next(19)
//S2 -> Event next(19)
```

```swift
    let connectable = Observable<Int>.interval(1, scheduler: ConcurrentDispatchQueueScheduler(qos: .background))
      .debug("interval 1")
      .multicast(ReplaySubject<Int>.create(bufferSize: 2))

    let refCount = connectable.refCount()

    refCount
      .debug("S1")
      .subscribe()
      .disposed(by: self.disposeBagS1)

    DispatchQueue.main
      .asyncAfter(deadline: DispatchTime.now() + 11.5) {
        self.disposeBagS1 = DisposeBag()
    }

    DispatchQueue.main
      .asyncAfter(deadline: DispatchTime.now() + 14.5) {
        refCount
          .debug("S2")
          .subscribe()
          .disposed(by: self.disposeBagS2)
    }
```
For the `refCount` there is some behaviour worth discussing. When the number of subscribers drops to zero, the underlying Observable terminates. But ReplaySubject remembers the last 2 items of the first Observable, and when we call subscribe, we create the second underlying Observable and at the same time we are  given the last 2 items of the first one:

```swift
//interval 1 -> Event next(7)
//S1 -> Event next(7)
//interval 1 -> Event next(8)
//S1 -> Event next(8)
//interval 1 -> Event next(9)
//S1 -> Event next(9)
//interval 1 -> Event next(10)
//S1 -> Event next(10)
//S1 -> isDisposed
//interval 1 -> isDisposed
//S2 -> subscribed
//S2 -> Event next(9)
//S2 -> Event next(10)
//interval 1 -> subscribed
//interval 1 -> Event next(0)
//S2 -> Event next(0)
//interval 1 -> Event next(1)
//S2 -> Event next(1)
//interval 1 -> Event next(2)
//S2 -> Event next(2)
//interval 1 -> Event next(3)
//S2 -> Event next(3)
```

### Share

We've discovered all possible cases for `connect`, `refCount` and `replay`. But there is one more case for specific `scope` argument of `share(replay: scope:)` function. If we call `share(replay: 0)`, which is the same as `share()`, the second argument doesn't have any effect. It affects behavior only when `replay > 0`.

Let's reproduce our last case 3, but replace `multicast`.`refCount` with:

#### `.share(replay: 2, scope: .forever)`

Which is:

```swift
self.multicast(ReplaySubject.create(bufferSize: replay)).refCount()
```

```swift
    let connectable = Observable<Int>.interval(1, scheduler: ConcurrentDispatchQueueScheduler(qos: .background))
      .debug("interval 1")
      .share(replay: 2, scope: .forever)

    connectable
      .debug("S1")
      .subscribe()
      .disposed(by: self.disposeBagS1)

    DispatchQueue.main
      .asyncAfter(deadline: DispatchTime.now() + 11.5) {
        self.disposeBagS1 = DisposeBag()
    }

    DispatchQueue.main
      .asyncAfter(deadline: DispatchTime.now() + 14.5) {
        connectable
          .debug("S2")
          .subscribe()
          .disposed(by: self.disposeBagS2)
    }
```

We've already seen the same log in the previous printing:

```swift
//interval 1 -> Event next(7)
//S1 -> Event next(7)
//interval 1 -> Event next(8)
//S1 -> Event next(8)
//interval 1 -> Event next(9)
//S1 -> Event next(9)
//interval 1 -> Event next(10)
//S1 -> Event next(10)
//S1 -> isDisposed
//interval 1 -> isDisposed
//S2 -> subscribed
//S2 -> Event next(9)
//S2 -> Event next(10)
//interval 1 -> subscribed
//interval 1 -> Event next(0)
//S2 -> Event next(0)
//interval 1 -> Event next(1)
//S2 -> Event next(1)
//interval 1 -> Event next(2)
//S2 -> Event next(2)
//interval 1 -> Event next(3)
//S2 -> Event next(3)
```

#### `.share(replay: 2, scope: .whileConnected)` 

Which is:

```swift
self.multicast(makeSubject: { ReplaySubject.create(bufferSize: replay) }).refCount()
```

```swift
    let connectable = Observable<Int>.interval(1, scheduler: ConcurrentDispatchQueueScheduler(qos: .background))
      .debug("interval 1")
      .share(replay: 2, scope: .whileConnected)

    connectable
      .debug("S1")
      .subscribe()
      .disposed(by: self.disposeBagS1)

    DispatchQueue.main
      .asyncAfter(deadline: DispatchTime.now() + 11.5) {
        self.disposeBagS1 = DisposeBag()
    }

    DispatchQueue.main
      .asyncAfter(deadline: DispatchTime.now() + 14.5) {
        connectable
          .debug("S2")
          .subscribe()
          .disposed(by: self.disposeBagS2)
    }
```

```swift
//interval 1 -> Event next(7)
//S1 -> Event next(7)
//interval 1 -> Event next(8)
//S1 -> Event next(8)
//interval 1 -> Event next(9)
//S1 -> Event next(9)
//interval 1 -> Event next(10)
//S1 -> Event next(10)
//S1 -> isDisposed
//interval 1 -> isDisposed
//S2 -> subscribed
//interval 1 -> subscribed
//interval 1 -> Event next(0)
//S2 -> Event next(0)
//interval 1 -> Event next(1)
//S2 -> Event next(1)
//interval 1 -> Event next(2)
//S2 -> Event next(2)
```

I prefer using this combination `multicast(makeSubject: { ReplaySubject.create(bufferSize: replay) }).refCount()` rather than `multicast(ReplaySubject.create(bufferSize: replay)).refCount()`. For me it makes much more sense.

And the comments for `scope` are also useful. 

`.whileConnected`:

* `retry` or `concat` operators will function as expected because terminating the sequence will clear the internal state.
* Each connection to the source observable sequence will use its own subject.
* When the number of subscribers drops from 1 to 0 and the connection to the source sequence is disposed, the subject will be cleared.

`.forever`:

* Using retry or concat operators after this operator isn’t usually advised.
* Each connection to the source observable sequence will share the same subject.
* After the number of subscribers drops from 1 to 0 and connection to the source observable sequence is disposed, this operator will continue holding a reference to the same subject. If at some later moment a new observer initiates a new connection to the source it can potentially receive some of the stale events received during the previous connection.
* After the source sequence terminates, any new observer will always immediately receive replayed elements and a terminal event. No new subscriptions to the source observable sequence will be attempted.

### Conclusion

Looks like there are too many unreal examples which aren't connected to RxSwift usage in real MVVM-like projects. But knowing the theory might help you avoid some mistakes in the future.

Let's take a look at some real examples / mistakes which could be found in real projects:

```swift
    let hotels: Observable<[Hotel]> = apiClient.hotels()
    let count = hotels.map { "We've found \($0.count) hotels" }
    let rating = hotels.map { "Average rating is \($0.reduce(0.0, { $0 + $1.rating }) / Float($0.count))" }

    count
      .bind(to: countLabel.rx.text)
      .disposed(by: disposeBag)

    rating
      .bind(to: averageRatingLabel.rx.text)
    .disposed(by: disposeBag)
```

Yep, you're right, all we need here is to avoid extra calls (we don't know whether `hotels()` Observable is hot or cold, but I guess for API implementation it should be cold) we need to fix `apiClient.hotels()` adding `share()` after it: 

```swift
    let hotels: Observable<[Hotel]> = apiClient.hotels().share()
    let count = hotels.map { "We've found \($0.count) hotels" }
    let rating = hotels.map { "Average rating is \($0.reduce(0.0, { $0 + $1.rating }) / Float($0.count))" }

    count
      .bind(to: countLabel.rx.text)
      .disposed(by: disposeBag)

    rating
      .bind(to: averageRatingLabel.rx.text)
    .disposed(by: disposeBag)
```

Another overhead I was fixing when working on my last project was the following problem:

```swift
  let hotelsBehavior = BehaviorRelay<[Hotel]>(value: [])
  lazy var hotelsObservable = hotelsBehavior.asObservable()

  lazy var hotelsCount = hotelsObservable
    .share()
    .map { $0.count }

  lazy var hotelsFound = hotelsObservable
    .share()
    .map { "We've found \($0.count) hotels" }

  lazy var averageRating = hotelsObservable
    .share()
    .map { $0.reduce(0.0, { $0 + $1.rating }) / Float($0.count) }
```

In the first example with `apiClient.hotels()` we weren't sure that Observable was hot. But in this example we see that `hotelsObservable` is just `BehaviorRelay`. Even without `share` for all observables when we subscribe for them we won't create 3 instances of `BehaviorRelay`. It should be shared for them.

When we apply `share` operator for each observable, we will create intermediate `PublishSubject` for each one. After subscription we'll have something like:

```swift
    hotelsCount
      .subscribe(onNext: { count in
        print(count)
      })
      .disposed(by: disposeBag)
      
    //is the same as
    let hotelsCountSubject = PublishSubject<[Hotel]>()

    hotelsBehavior
      .bind(to: hotelsCountSubject)
      .disposed(by: disposeBag)

    hotelsCountSubject
      .map { $0.count }
      .subscribe(onNext: { count in
        print(count)
      })
      .disposed(by: disposeBag)
```

Looks like a resource being wasted in the project. To fix the problem all we need is just to remove extra `share` calls.
