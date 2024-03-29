---
title: "Filtering Category"
date: 2019-06-28 12:00:00 +0300
categories: rxswift
toc: true
toc_sticky: true
---

## Filtering Category

Today I'm going thoroughly through all the available operators in the filtering category from the [reactivex](http://reactivex.io/documentation/operators.html#filtering).

### Filter

This operator is very similar to the standard `filter` method which works with collections, moreover, syntax is the same. The only difference is that it's an RxSwift operator, it ignores all events which don't meet the condition.

```swift
Observable
  .of(0, 1, 1, 2, 3, 5, 8, 13, 21, 34, 55)
  .filter { $0 > 5 }
  .debug("numbers greater than 5")
  .subscribe()
  .disposed(by: disposeBag)
```

```swift
numbers -> subscribed
numbers -> Event next(8)
numbers -> Event next(13)
numbers -> Event next(21)
numbers -> Event next(34)
numbers -> Event next(55)
numbers -> Event completed
numbers -> isDisposed
```

### Take

#### Take count

We've already seen this operator in action (`.take(1).asSingle()`), when I described [what RxSwift traits]({% post_url 2019-05-17-traits %}) are. It takes `count` number of `next` events from the source observable and emits `completed` after them.

```swift
Observable
  .of(0, 1, 1, 2, 3, 5, 8, 13, 21, 34, 55)
  .take(5)
  .debug("take 5")
  .subscribe()
  .disposed(by: disposeBag)
```

```swift
take 5 -> subscribed
take 5 -> Event next(0)
take 5 -> Event next(1)
take 5 -> Event next(1)
take 5 -> Event next(2)
take 5 -> Event next(3)
take 5 -> Event completed
take 5 -> isDisposed
```

#### Take last

It returns a specified number of contiguous elements from the end of an observable sequence. It works similarly as the take count operator when the sequence completes at some point.

```swift
Observable
  .of(0, 1, 1, 2, 3, 5, 8, 13, 21, 34, 55)
  .takeLast(5)
  .debug("take last 5")
  .subscribe()
  .disposed(by: disposeBag)
```

```swift
take last 5 -> subscribed
take last 5 -> Event next(8)
take last 5 -> Event next(13)
take last 5 -> Event next(21)
take last 5 -> Event next(34)
take last 5 -> Event next(55)
take last 5 -> Event completed
take last 5 -> isDisposed
```

But **be careful**, for an infinity sequence this operator will never emit any event, like in this case:

```swift
Observable<Int>
  .interval(1, scheduler: MainScheduler.instance)
  .takeLast(5)
  .debug("take last 5")
  .subscribe()
  .disposed(by: disposeBag)
```

```swift
take last 5 -> subscribed
```

#### TakeWhile

Returns elements from an observable sequence as long as a specified condition is `true`. Once the condition is false, the result observable completes.

Let's discuss the following imaginary situation. We've got a sequence that emits Int events in the following repeated manner: 

[0, 1, 2, 3, 4, 0, 1, 2, 3, 4, 0, 1, 2, 3, 4, ...]

```swift
Observable<Int>
  .interval(0.5, scheduler: MainScheduler.instance)
  .map { num -> Int in
    if num >= 5 {
      return num % 5
    } else {
      return num
    }
  }
  .takeWhile { $0 < 4 }
  .debug("takeWhile < 4")
  .subscribe()
  .disposed(by: disposeBag)
```

```swift
takeWhile < 4 -> subscribed
takeWhile < 4 -> Event next(0)
takeWhile < 4 -> Event next(1)
takeWhile < 4 -> Event next(2)
takeWhile < 4 -> Event next(3)
takeWhile < 4 -> Event completed
takeWhile < 4 -> isDisposed
```

If the source observable emits events that always meet the condition in the `takeWhile` closure (for example, `.takeWhile { $0 < 5 }` in the example above), the result observable will repeat the source behavior.

#### Take duration

It takes elements for the specified duration from the start of the observable source sequence, using the specified scheduler to run timers.

```swift
Observable<Int>
  .interval(1, scheduler: MainScheduler.instance)
  .take(5, scheduler: MainScheduler.instance)
  .debug("take duration = 5")
  .subscribe()
  .disposed(by: disposeBag)
```

```swift
take duration = 5 -> subscribed
take duration = 5 -> Event next(0)
take duration = 5 -> Event next(1)
take duration = 5 -> Event next(2)
take duration = 5 -> Event next(3)
take duration = 5 -> Event completed
take duration = 5 -> isDisposed
```

#### Take until

It returns the elements from the source observable sequence until the other observable sequence produces an element.

```swift
let stopSequence = Observable<Int>
  .just(1)
  .delay(5, scheduler: MainScheduler.instance)

Observable<Int>
  .interval(1, scheduler: MainScheduler.instance)
  .takeUntil(stopSequence)
  .debug("take until")
  .subscribe()
  .disposed(by: disposeBag)
```

```swift
take until -> subscribed
take until -> Event next(0)
take until -> Event next(1)
take until -> Event next(2)
take until -> Event next(3)
take until -> Event completed
take until -> isDisposed
```

### Skip

#### Skip count

It bypasses a specified number of elements in an observable sequence and then returns the remaining elements.

```swift
Observable<Int>
  .interval(1, scheduler: MainScheduler.instance)
  .debug("source")
  .skip(4)
  .debug("result after skip 4")
  .subscribe()
  .disposed(by: disposeBag)
```

```swift
result after skip 4 -> subscribed
source -> subscribed
source -> Event next(0)
source -> Event next(1)
source -> Event next(2)
source -> Event next(3)
//four events above were skipped
source -> Event next(4)
result after skip 4 -> Event next(4)
source -> Event next(5)
result after skip 4 -> Event next(5)
source -> Event next(6)
result after skip 4 -> Event next(6)
```

#### Skip duration 

It skips elements for the specified duration from the start of the observable source sequence, using the specified scheduler to run timers.

```swift
Observable<Int>
  .interval(0.5, scheduler: MainScheduler.instance)
  .debug("source")
  .skip(3, scheduler: MainScheduler.instance)
  .debug("result skip duration 5")
  .subscribe()
  .disposed(by: disposeBag)
```

```swift
result skip duration 5 -> subscribed
source -> subscribed
source -> Event next(0)
source -> Event next(1)
source -> Event next(2)
source -> Event next(3)
source -> Event next(4)
source -> Event next(5)
result skip duration 5 -> Event next(5)
source -> Event next(6)
result skip duration 5 -> Event next(6)
source -> Event next(7)
result skip duration 5 -> Event next(7)
```

#### Skip while

It bypasses elements in an observable sequence as long as a specified condition is true and then returns the remaining elements.

```swift
Observable<Int>
  .interval(0.5, scheduler: MainScheduler.instance)
  .map { num -> Int in
    if num >= 5 {
      return num % 5
    } else {
      return num
    }
  }
  .skipWhile { $0 < 4 }
  .debug("skipWhile < 4")
  .subscribe()
  .disposed(by: disposeBag)
```

```swift
skipWhile < 4 -> subscribed
skipWhile < 4 -> Event next(4)
skipWhile < 4 -> Event next(0)
skipWhile < 4 -> Event next(1)
skipWhile < 4 -> Event next(2)
skipWhile < 4 -> Event next(3)
skipWhile < 4 -> Event next(4)
skipWhile < 4 -> Event next(0)
skipWhile < 4 -> Event next(1)
skipWhile < 4 -> Event next(2)
skipWhile < 4 -> Event next(3)
```

#### Skip until

It returns the elements from the source observable sequence that are emitted after the other observable sequence produces an element.

```swift
let startSequence = Observable<Int>
  .just(1)
  .delay(5, scheduler: MainScheduler.instance)

Observable<Int>
  .interval(1, scheduler: MainScheduler.instance)
  .skipUntil(startSequence)
  .debug("skipUntil startSequence")
  .subscribe()
  .disposed(by: disposeBag)
```

```swift
skipUntil startSequence -> subscribed
skipUntil startSequence -> Event next(5)
skipUntil startSequence -> Event next(6)
skipUntil startSequence -> Event next(7)
skipUntil startSequence -> Event next(8)
```

#### Skip vs Take

In general, `skip` is the opposite of `take`. If they were applied to the same source with the same arguments, they would "split" the source into the same element.

Let's take a look one more time at the source:

[0, 1, 2, 3, 4, 0, 1, 2, 3, 4, 0, 1, 2, 3, 4, ...] - source

[**0, 1, 2, 3**, 4, 0, 1, 2, 3, 4, 0, 1, 2, 3, 4, ...] - `takeWhile`

[0, 1, 2, 3, **4, 0, 1, 2, 3, 4, 0, 1, 2, 3, 4, ...**] - `skipWhile`

### Distinct

`distinctUntilChanged` operator returns an observable sequence that contains only distinct contiguous elements according to the equality operator.

```swift
Observable
  .of(0, 0, 0, 1, 0, 0, 0, 0, 0, 1)
  .distinctUntilChanged()
  .debug("distinctUntilChanged")
  .subscribe()
  .disposed(by: disposeBag)
```

```swift
distinctUntilChanged -> subscribed
distinctUntilChanged -> Event next(0)
distinctUntilChanged -> Event next(1)
distinctUntilChanged -> Event next(0)
distinctUntilChanged -> Event next(1)
distinctUntilChanged -> Event completed
distinctUntilChanged -> isDisposed
```

As you can see, event isn't forwarded if the previous event is the same.

### Debounce and Throttle

Let's discuss the common scenario: a user types something in the search field, for each change an API call is performed. After `debounce` or `throttle`, the number of API call is reduced and it depends on delays.

#### Debounce

It ignores elements from an observable sequence which are followed by another element within a specified relative time duration, using the specified scheduler to run throttling timers.

The following example simulates quick users' input, but from number 5 to 10 it slows down a bit.

```swift
Observable<Int>
  .interval(0.2, scheduler: MainScheduler.instance)
  .filter { num in
    if num > 5 && num < 10 {
      return num % 2 == 0
    } else {
      return true
    }
  }
  .debug("source")
  .debounce(0.3, scheduler: MainScheduler.instance)
  .debug("debounce applied")
  .subscribe()
  .disposed(by: disposeBag)
```

```swift
debounce applied -> subscribed
source -> subscribed
source -> Event next(0)
source -> Event next(1)
source -> Event next(2)
source -> Event next(3)
source -> Event next(4)
source -> Event next(5)
source -> Event next(6)
debounce applied -> Event next(6)
source -> Event next(8)
debounce applied -> Event next(8)
source -> Event next(10)
source -> Event next(11)
source -> Event next(12)
```

We captured exactly these "numbers" when the user types slowly (0.4) then usual (0.2).

#### Throttle

It returns an Observable that emits the first and the latest item emitted by the source Observable during sequential time windows of a specified duration.

```swift
Observable<Int>
  .interval(0.2, scheduler: MainScheduler.instance)
  .debug("source")
  .throttle(1, scheduler: MainScheduler.instance)
  .debug("throttle applied")
  .subscribe()
  .disposed(by: disposeBag)
```

```swift
throttle applied -> subscribed
source -> subscribed
source -> Event next(0)
throttle applied -> Event next(0)
source -> Event next(1)
source -> Event next(2)
source -> Event next(3)
source -> Event next(4)
source -> Event next(5)
throttle applied -> Event next(5)
source -> Event next(6)
source -> Event next(7)
source -> Event next(8)
source -> Event next(9)
source -> Event next(10)
throttle applied -> Event next(10)
```

### ElementAt

It returns a sequence emitting only element _n_ emitted by an Observable.

```swift
Observable<Int>
  .interval(0.2, scheduler: MainScheduler.instance)
  .debug("source")
  .elementAt(5)
  .debug("elementAt 5")
  .subscribe()
  .disposed(by: disposeBag)
```

```swift
elementAt 5 -> subscribed
source -> subscribed
source -> Event next(0)
source -> Event next(1)
source -> Event next(2)
source -> Event next(3)
source -> Event next(4)
source -> Event next(5)
elementAt 5 -> Event next(5)
elementAt 5 -> Event completed
elementAt 5 -> isDisposed
source -> isDisposed
```

### IgnoreElements

It skips elements and completes (or errors) when the observable sequence completes (or errors). The equivalent to `filter` operator that always returns false. It will never completes, if the source never completes.

```swift
Observable<Int>
  .interval(0.2, scheduler: MainScheduler.instance)
  .take(5)
  .debug("source")
  .ignoreElements()
  .debug("ignoreElements")
  .subscribe()
  .disposed(by: disposeBag)
```

```swift
ignoreElements -> subscribed
source -> subscribed
source -> Event next(0)
source -> Event next(1)
source -> Event next(2)
source -> Event next(3)
source -> Event next(4)
source -> Event completed
source -> isDisposed
ignoreElements -> Event completed
ignoreElements -> isDisposed
```

### Sample

It samples the source observable sequence using a sampler observable sequence producing sampling ticks. Upon each sampling tick, the latest element (if any) in the source sequence during the last sampling interval is sent to the resulting sequence.

```swift
let sampler = Observable<Int>
  .interval(1, scheduler: MainScheduler.instance)

Observable<Int>
  .interval(0.3, scheduler: MainScheduler.instance)
  .debug("source")
  .sample(sampler)
  .debug("sample applied")
  .subscribe()
  .disposed(by: disposeBag)
```

```swift
sample applied -> subscribed
source -> subscribed
source -> Event next(0)
source -> Event next(1)
source -> Event next(2)
sample applied -> Event next(2)
source -> Event next(3)
source -> Event next(4)
source -> Event next(5)
sample applied -> Event next(5)
source -> Event next(6)
source -> Event next(7)
source -> Event next(8)
source -> Event next(9)
sample applied -> Event next(9)
```
