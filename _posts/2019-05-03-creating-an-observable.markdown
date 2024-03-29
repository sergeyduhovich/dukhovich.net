---
title: "Creating an Observable"
date: 2019-05-03 12:00:00 +0300
categories: rxswift
toc: true
toc_sticky: true
---

In this post I'm going to run through all methods available in  the [Create](http://reactivex.io/documentation/operators.html#creating) section on reactivex website.

## Create

It creates an Observable from scratch by calling observer methods programmatically. This method is commonly used to transform a usual closure to its Observable equivalent:

```swift
struct Hotel: Codable {
  let title: String
  let rating: Float
}

enum APIError: Error {
  case noResponse
  case invalidFormat
  case invalidEndpoint
}

let intObservable = Observable<Int>.create { observer -> Disposable in
  observer.onNext(1)
  observer.onNext(2)
  observer.onCompleted()
  return Disposables.create()
}

let hotelsObservable = Observable<[Hotel]>.create { observer -> Disposable in
  guard let url = URL(string: "https://cool.api.hotels.com/v1/hotels") else {
    observer.onError(APIError.invalidEndpoint)
    return Disposables.create()
  }
  let task = URLSession.shared
    .dataTask(with: url) { (data, response, error) in
      do {
        guard let data = data else {
          observer.onError(APIError.noResponse)
          return
        }
        let json = try JSONDecoder().decode([Hotel].self, from: data)
        observer.onNext(json)
        observer.onCompleted()
      } catch {
        observer.onError(error)
      }
  }
  
  task.resume()
  
  return Disposables.create {
    task.cancel()
  }
}

intObservable
  .subscribe(onNext: { number in
    print(number)
  })
  .disposed(by: disposeBag)

hotelsObservable
  .subscribe(onNext: { hotels in
    print(hotels)
  })
  .disposed(by: disposeBag)
```

## Defer

It doesn't create the Observable until the observer subscribes, and create a fresh 
Observable for each observer.

Let's take a look at the following problem:

```swift
enum MyError: Error {
  case invalidCondition(String)
}

class ClassWithHeavyInit {
  init?() {
    print("initializer")
    sleep(5)
    print("initializer after sleep")
    if Int.random(in: 0...5) != 5 {
      return nil
    }
  }
}

func slowObservable() -> Observable<ClassWithHeavyInit> {
  if let result = ClassWithHeavyInit() { //executed without subscribers
  	//executed only with subscribers
    return .just(result)
  } else {
  	//executed only with subscribers
    return .error(MyError.invalidCondition(String(describing: ClassWithHeavyInit.self))) //executed only with subscribers
  }
}

slowObservable()
```

Even if there are no subscribers, all the code except `return .just(...)` or `return .error(...)` will be executed. And we can see it in the console:

```
initializer
initializer after sleep
```

To avoid allocating resources before they are really needed we could "wrap" our method in `deferred` closure. Let's take a look at the new `slowObservable` fixed implementation:

```swift
func slowObservable() -> Observable<ClassWithHeavyInit> {
  return Observable.deferred {
    if let result = ClassWithHeavyInit() {
      return .just(result)
    } else {
      return .error(MyError.invalidCondition(String(describing: ClassWithHeavyInit.self)))
    }
  }
}
```

And there is nothing in the console until we actually subscribe to `slowObservable()`.

I found it useful when working with [Action](https://github.com/RxSwiftCommunity/Action) extension, especially where I had to deal with a lot of returns like `return .just(smth)`.

## Empty

It creates an empty observable which immediately completes:

```swift
_ = Observable<Int>
  .empty()
  .debug("empty")
  .subscribe()
```

it prints:

```swift
empty -> subscribed
empty -> Event completed
empty -> isDisposed
```

Another common scenario for using `empty` is the following:

```swift
class ViewController: UIViewController {

  override func viewDidLoad() {
    super.viewDidLoad()

    viewModel.hotels
      .flatMap { [weak self] hotels -> Observable<Float> in
        guard let self = self else { return Observable<Float>.empty() }
        return self.calculateRating(hotels: hotels)
      }
      .subscribe(onNext: { rating in
        print(rating)
      })
      .disposed(by: disposeBag)
  }
  
  func calculateRating(hotels: [Hotel]) -> Observable<Float> { ... }
}
```

When we need to call some method on self inside a closure, and `self` is actually in a capture list and weak. While guarding it to strong we can make a compiler happy by returning `.empty()`. No value is required.

## Never

It creates an infinity observable which never completes.

```swift
Observable<Int>
  .never()
  .debug("never")
  .subscribe()
  .disposed(by: disposeBag)
```

it prints:

```swift
never -> subscribed
```

Be careful with this method. For `empty` we don't need a `disposeBag` to free the allocated memory, cause it completes immediately. `never` is the opposite, you always have to add it to some `disposeBag` to avoid memory leaks. 

I don't see too many suitable cases for this operator. A possible scenario is when we have to conform some protocol in the Test target to create a mock Class, in this case we could use it as the simplest return.

## Error

It creates an observable which emits error and terminates.

```swift
enum MyError: Error {
  case customError
}

_ = Observable<Int>
  .error(MyError.customError)
  .debug("error")
  .subscribe()
```

it prints:

```swift
error -> subscribed
error -> Event error(customError)
error -> isDisposed
```

Let's take a look one more time at the example used in the Defer section:

```swift
func slowObservable() -> Observable<ClassWithHeavyInit> {
  if let result = ClassWithHeavyInit() {
    return .just(result)
  } else {
    return .error(MyError.invalidCondition(String(describing: ClassWithHeavyInit.self)))
  }
}
```

## From 

It converts some other object or data structure into an Observable. There are 3 possible first arguments: an `Array`,a `Sequence` and an `Optional`, the second argument, `scheduler`, is optional.

For arrays and sequences:

```swift
Observable.from([1, 2, 3, 4], scheduler: MainScheduler.instance)
  .debug("from array")
  .subscribe()
  .disposed(by: disposeBag)
```

it prints:

```swift
from array -> subscribed
from array -> Event next(1)
from array -> Event next(2)
from array -> Event next(3)
from array -> Event next(4)
from array -> Event completed
from array -> isDisposed
```

The optional argument converts any object into an Observable sequence similar to `.just` operator, but there is a possibility to pass the scheduler:

```swift
Observable.from(optional: "QWERTY", scheduler: MainScheduler.instance)
  .debug("from qwerty")
  .subscribe()
  .disposed(by: disposeBag)
```

it prints:

```swift
from qwerty -> subscribed
from qwerty -> Event next(QWERTY)
from qwerty -> Event completed
from qwerty -> isDisposed
```

Things change a bit when we pass nil as the first argument. It starts working like `.empty` operator:

```swift
Observable<Int>.from(optional: nil, scheduler: MainScheduler.instance)
  .debug("from nil")
  .subscribe()
  .disposed(by: disposeBag)
```

it prints:

```swift
from nil -> subscribed
from nil -> Event completed
from nil -> isDisposed
```

Another way to convert a sequence to an Observable is `of` operator

```swift
Observable.of([1, 2, 3, 4])
  .debug("of array")
  .subscribe()
  .disposed(by: disposeBag)
```

it prints:

```swift
of array -> subscribed
of array -> Event next([1, 2, 3, 4])
of array -> Event completed
of array -> isDisposed
```

In comparison with `from` printing, we see only one next event, while `from` emits the same number of the next events as array count.

## Interval 

It creates an Observable that emits a sequence of integers spaced by a particular time interval:

```swift
Observable<Int>.interval(0.5, scheduler: ConcurrentDispatchQueueScheduler(qos: .background))
  .debug("background")
  .subscribe()
  .disposed(by: disposeBag)

Observable<Int>.interval(1, scheduler: MainScheduler.instance)
  .debug("UI")
  .subscribe()
  .disposed(by: disposeBag)

sleep(3)
```

it starts printing until you kill the app:

```swift
background -> subscribed
UI -> subscribed
background -> Event next(0)
background -> Event next(1)
background -> Event next(2)
background -> Event next(3)
background -> Event next(4)
UI -> Event next(0)
background -> Event next(5)
background -> Event next(6)
UI -> Event next(1)
background -> Event next(7)
background -> Event next(8)
background -> Event next(9)
UI -> Event next(2)
background -> Event next(10)
background -> Event next(11)
UI -> Event next(3)
...
```

## Just 

It converts an object or a set of objects into an Observable which emits that or those objects:

```swift
let just = Observable<Int>.just(1)

let justBg = Observable<Int>.just(1, scheduler: ConcurrentDispatchQueueScheduler(qos: .background))
```

## Range 

It creates an Observable that emits a range of sequential integers:

```swift
Observable<Int>.range(start: 1, count: 7, scheduler: ConcurrentDispatchQueueScheduler(qos: .background))
  .debug("background")
  .subscribe()
  .disposed(by: disposeBag)

Observable<Int>.range(start: 1, count: 7, scheduler: MainScheduler.instance)
  .debug("UI")
  .subscribe()
  .disposed(by: disposeBag)

sleep(3)
```

it prints:

```swift
background -> subscribed
background -> Event next(1)
UI -> subscribed
UI -> Event next(1)
background -> Event next(2)
background -> Event next(3)
background -> Event next(4)
background -> Event next(5)
background -> Event next(6)
background -> Event next(7)
background -> Event completed
background -> isDisposed

//and after few seconds

UI -> Event next(2)
UI -> Event next(3)
UI -> Event next(4)
UI -> Event next(5)
UI -> Event next(6)
UI -> Event next(7)
UI -> Event completed
UI -> isDisposed
```

There is a strange issue with UI first next event. We expect that Main queue is blocked by `sleep(3)`

## Repeat 

It creates an Observable that emits a particular item or sequence of items repeatedly:

```swift
Observable<Int>.repeatElement(1)
  .debug("UI")
  .subscribe()
  .disposed(by: disposeBag)

Observable<String>.repeatElement("bla", scheduler: ConcurrentDispatchQueueScheduler(qos: .background))
  .debug("background")
  .subscribe()
  .disposed(by: disposeBag)
```

it prints untill you kill the app:

```swift
UI -> subscribed
UI -> Event next(1)
UI -> Event next(1)
UI -> Event next(1)
UI -> Event next(1)
UI -> Event next(1)
UI -> Event next(1)
UI -> Event next(1)
UI -> Event next(1)
...
```

As you can see, there are no prints from the second Observable, it's because we can't reach the 6th line of code, cause we did blocked Main thread. I don't see where I could use this operator, for me it looks quite useless, especially on Main thread, very similar to `while true { }`

## Timer

It creates an Observable that emits a single item after a given delay:

```swift
Observable<Int>.timer(5, scheduler: MainScheduler.instance)
  .debug("once after 5s")
  .subscribe()
  .disposed(by: disposeBag)

Observable<Int>.timer(3, period: 0.5, scheduler: MainScheduler.instance)
  .debug("UI")
  .subscribe()
  .disposed(by: disposeBag)

Observable<Int>.timer(1, period: 0.5, scheduler: ConcurrentDispatchQueueScheduler(qos: .background))
  .debug("background")
  .subscribe()
  .disposed(by: disposeBag)
```

it prints:

```swift
once after 5s -> subscribed
UI -> subscribed
background -> subscribed
background -> Event next(0)
background -> Event next(1)
background -> Event next(2)
background -> Event next(3)
UI -> Event next(0)
background -> Event next(4)
UI -> Event next(1)
background -> Event next(5)
UI -> Event next(2)
background -> Event next(6)
UI -> Event next(3)
background -> Event next(7)
once after 5s -> Event next(0)
once after 5s -> Event completed
once after 5s -> isDisposed
UI -> Event next(4)
background -> Event next(8)
UI -> Event next(5)
background -> Event next(9)
UI -> Event next(6)
background -> Event next(10)
```

If there is no `period` parameter, it works like delay, if there is a `period` parameter it works like delayed `interval`, and a scheduler parameter is available too. 
