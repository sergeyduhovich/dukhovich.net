---
title: "Error handling"
date: 2019-06-07 12:00:00 +0300
categories: rxswift
toc: true
toc_sticky: true
---

**Normal termination** of an Observable sequence is **Completed** or **Error** event. After one of them is emitted, Observable can’t emit any Next event and resources for the Observable are released.

The paragraph above is very important in terms of ReactiveX contract. But at the time we start writing Rx-code, we could forget this contract. At first glance the following example looks fine:

```swift
class ViewController: UIViewController {
	let apiCallObservable: Observable<Int>
	//...
	override func viewDidLoad() {
		super.viewDidLoad()
		
	    apiCallObservable
		  .map { "We've found \($0) hotels for you" }
		  .bind(to: label.rx.text)
		  .disposed(by: disposeBag)
	}
}
```

We've got an Observable which represents an API call which returns a number of hotels. The result is transformed and bound to the text property of `label`.

What if we received an error? The answer is simple: the Observable would terminate immediately, resources would be released and label wouldn't be updated.

For error handling I use the following operators and it depends on a scenario when they are applied:

* `materialize` and `dematerialize`;
* `catch`;
* `retry`;
* `retryWhen`;

I'll use this Observable source for all my samples:

```swift
let chatObservable: Observable<String> = Observable<Int>
  .interval(1, scheduler: MainScheduler.instance)
  .flatMap { num -> Observable<String> in
    if Int.random(in: 0..<3) == 0 {
      return .error(DataError.testError)
    } else {
      return .just("message \(num)")
    }
}
```

It gives me an Observable that emits some events then errors out. We'll try to handle it properly.

## 1. Inner Observable errors out

Let's start with the simplest case. We have an Observable that never fails like button clicks ([ControlEvent trait]({% post_url 2019-05-17-traits %})). A click Observable is a source for the inner Observable applying `flatMap` operator. It means, that for each button click a new inner Observable will be created. The inner Observable forwards `error` and `next` events to the source. I like the fact that `completed` event doesn't affect the source. It gives us an option to transform `error` into `completed`, so I use this trick quite often:

### The problem

```swift
button.rx.tap.debug("source")
  .flatMapLatest { _ -> Observable<String> in
    return chatObservable.debug("inner")
  }
  .debug("result")
  .bind(to: label.rx.text)
  .disposed(by: disposeBag)
```

Without handling errors, the console output might look like this:

```swift
result -> subscribed
source -> subscribed
//after button click
source -> Event next(())
inner -> subscribed
inner -> Event next(message 0)
result -> Event next(message 0)
inner -> Event next(message 1)
result -> Event next(message 1)
inner -> Event next(message 2)
result -> Event next(message 2)
inner -> Event next(message 3)
result -> Event next(message 3)
inner -> Event error(testError)
result -> Event error(testError)
result -> isDisposed
source -> isDisposed
inner -> isDisposed
```

The problem here is that not only `result` Observable is disposed, but `source` too. That means we will not react to button clicks anymore.

### `catchError`

Ok, let's try to modify the inner Observable transforming an `error` into `completed` using `catchError` operator:

```swift
button.rx.tap.debug("source")
  .flatMapLatest { _ -> Observable<String> in
    return chatObservable.debug("inner")
      .catchError { [weak self] error -> Observable<String> in
        self?.label.text = "something went wrong: \(error)"
        return .empty()
      }
  }
  .debug("result")
  .bind(to: label.rx.text)
  .disposed(by: disposeBag)
```

After these changes a source Observable behaves as expected and never completes. Error has been handled, inner subscription is recreated for the following button clicks:

```swift
result -> subscribed
source -> subscribed
//after button click
source -> Event next(())
inner -> subscribed
inner -> Event next(message 0)
result -> Event next(message 0)
inner -> Event next(message 1)
result -> Event next(message 1)
inner -> Event next(message 2)
result -> Event next(message 2)
inner -> Event error(testError)
inner -> isDisposed
//result and source subscriptions are still alive
```

![error handling in rxswift 1](http://dukhovich.by/assets/images/articles/error_handling_rxswift.gif)

### `materialize` with following `dematerialize`

The same behavior as showed above could be written with `materialize` and `dematerialize` [operators](http://reactivex.io/documentation/operators/materialize-dematerialize.html). After `materialize` is applied, Observable emits all events, as `next` events are associated with the `Event` type. It doesn't mean that the source Observable continues working even if it errors out. No, the source Observable forwards an error as `.next(Event.error(testError))` and completes as expected.

Anyway, you might use `materialize` + `dematerialize` instead of `catchError` for the inner Observable if you like it. The subscription above might be changed to:

```swift
button.rx.tap.debug("source")
  .flatMapLatest { _ -> Observable<String> in
    return chatObservable.debug("inner")
      .materialize()
      .map { [weak self] event -> Event<String> in
        if case let .error(err) = event {
          self?.label.text = "something went wrong: \(error)"
          return .completed
        } else {
          return event
        }
      }
      .dematerialize()
  }
  .debug("result")
  .bind(to: label.rx.text)
  .disposed(by: disposeBag)
```

## 2. Original Observable errors out

### `retry`

To recover an original Observable after it errors out, there is the operator `retry`. In RxSwift it is represented by `retry(_ maxAttemptCount: Int)` and `retry()`. These operators could help you recreate a failed Observable saving the original subscription. You should be careful using it. Retrying an API call when there is no internet connection leads to resource and processor time wasting.

To be honest, I don't like this operator. If it is really needed, I use `rety` [from RxSwiftExt repo](https://github.com/RxSwiftCommunity/RxSwiftExt#retry), it has some options for retrying, like `exponentialDelayed` or `customTimerDelayed`.

### `retryWhen`

A more interactive retry is `retryWhen`. The example above could be transformed in the following way. We retry only after a user clicks on "retry" button:

```swift
chatObservable
  .debug("chatObservable")
  .retryWhen { [weak self] error -> Observable<Void> in
    return error.flatMapLatest { error -> Observable<Void> in
      guard let self = self else { return .empty() }
      self.label.text = "something went wrong: \(error)"
      return self.button.rx.tap.asObservable()
    }
  }
  .debug("result")
  .bind(to: label.rx.text)
  .disposed(by: disposeBag)
```

## RxCocoa

When working with UI we use a lot of [Driver and Signal traits]({% post_url 2019-05-17-traits %}). They could be created from a raw failable Observable with methods which are using `catch` under the hood.

### Driver:

```swift
public func asDriver(onErrorJustReturn: E) -> Driver<E>
public func asDriver(onErrorDriveWith: Driver<E>) -> Driver<E>
public func asDriver(onErrorRecover: @escaping (_ error: Swift.Error) -> Driver<E>) -> Driver<E>
```

### Signal:

```swift
public func asSignal(onErrorJustReturn: E) -> Signal<E>
public func asSignal(onErrorSignalWith: Signal<E>) -> Signal<E>
public func asSignal(onErrorRecover: @escaping (_ error: Swift.Error) -> Signal<E>) -> Signal<E>
```

## RxSwiftExt

There are some useful operators for error handling that could be found in [RxSwiftExt repository](https://github.com/RxSwiftCommunity/RxSwiftExt):

* [retry](https://github.com/RxSwiftCommunity/RxSwiftExt#retry)
* [errors, elements](https://github.com/RxSwiftCommunity/RxSwiftExt#errors-elements)
* [catcherrorjustcomplete](https://github.com/RxSwiftCommunity/RxSwiftExt#catcherrorjustcomplete)

## Summary

There are 2 distinctive strategies to recover an Observable sequence - **Catch** and **Retry**

Failable Observable as a source must be processed with `retry` operators. 

As soon as Observable is used inside `flatMap` it might be handled using all the operators from this article. It's up to you which one to pick.
