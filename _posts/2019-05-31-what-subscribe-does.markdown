---
title: "What subscribe does"
date: 2019-05-31 12:00:00 +0300
categories: rxswift
toc: true
toc_sticky: true
---

It doesn't matter how many operators you have to use to describe an Observable. Without subscription it's just a definition. The state of application will never change. How does subscription change an Observable from just a definition to an actual process? And why does it require us to add one more method call after `subscribe` call, usually it's `.disposed(by: disposeBag)`? 

Let's start and check what `DisposeBag` is?

```swift
//DisposeBag.swift (simplified)
public final class DisposeBag: DisposeBase {
	fileprivate var _disposables = [Disposable]()
	
	public func insert(_ disposable: Disposable) {
		self._disposables.append(disposable)
	}
	    
	deinit {
	    for disposable in _disposables {
	        disposable.dispose()
	    }
	}
}
```

It's a container for `Disposable` objects. As soon as the container deallocates, the `dispose()` method is called on all objects in it. This method in actual implementation of `Disposable` protocol should dispose the retained resources.

Even if `DisposeBag` and `Disposable` classes might seem clear, the whole process under `subscribe` might not. To understand what subscription is and what is going on there we have to dive deeper in the sources of RxSwift. Let's check out the implementation of the  `subscribe` method:

```swift
//Observable.swift
public class Observable<Element> : ObservableType {
	public func subscribe<O: ObserverType>(_ observer: O) -> Disposable where O.E == E {
		rxAbstractMethod()
	}
}

//Rx.swift
func rxAbstractMethod(file: StaticString = #file, line: UInt = #line) -> Swift.Never {
	rxFatalError("Abstract method", file: file, line: line)
}
```

`rxAbstractMethod` forces subclasses to implement this method, otherwise it crashes the app.

### Observable

After some research, I collected classes that override the `subscribe` method. On the following UML diagram, you could find these classes. Moreover, I tried to display the connections with the parent classes and protocols:

![img](http://dukhovich.by/assets/images/articles/subscribe_diagram.png)

There are a lot of classes with varied responsibilities, some of them are part of RxSwift core library, the others come from RxCocoa and RxRelay. Let's focus now on `Observable` subclasses. There are 3 vivid groups: producer + subclasses, subjects and classes which are used to [share computation resources]({% post_url 2019-05-10-connectable-observable %}). The easiest place to start our investigation journey is `Producer` and its subclasses. They implement simple 1-to-1 subscription logic without any resource sharing.

Let's take a look at `Producer.swift` file(shortened):

```swift
class Producer<Element> : Observable<Element> {
    override func subscribe<O : ObserverType>(_ observer: O) -> Disposable where O.E == Element {
        let disposer = SinkDisposer()
        let sinkAndSubscription = self.run(observer, cancel: disposer)
        disposer.setSinkAndSubscription(sink: sinkAndSubscription.sink, subscription: sinkAndSubscription.subscription)
        return disposer
    }

    func run<O : ObserverType>(_ observer: O, cancel: Cancelable) -> (sink: Disposable, subscription: Disposable) where O.E == Element {
        rxAbstractMethod()
    }
}
```

### Subscribe

Ok, what is going on in the `subscribe` method? Let's read the code line by line.

1. It creates a `SinkDisposer` object;
2. It passes this disposer and observer to `run` method which returns the tuple of two `Disposable` instances;
3. These 2 disposables (sink and subscription) are passed to `setSinkAndSubscription` method of the disposer;
4. It returns the disposer;

#### SinkDisposer

`SinkDisposer` conforms `Cancelable` which conforms `Disposable`:

```swift
fileprivate final class SinkDisposer: Cancelable {
    private var _sink: Disposable?
    private var _subscription: Disposable?

    func setSinkAndSubscription(sink: Disposable, subscription: Disposable) {
        self._sink = sink
        self._subscription = subscription
        //...
    }
    
    func dispose() {
		//...
	    sink.dispose()
	    subscription.dispose()
	
	    self._sink = nil
	    self._subscription = nil
	}
}
```

The responsibility of `setSinkAndSubscription` method of `SinkDisposer` is simple, it retains both references, sink and subscription. And the class itself is an actual implementation of `Disposable` protocol, which we see every time we call `subscribe`. And each time you call `dispose()` on the subscription result, you call the method on `SinkDisposer` instance.

#### Run

Producer requires subclasses to override this method. To understand what happens here I'm going to check 3 particular subclasses of `Producer`- 3 operators. They are `interval`, `filter` and `skip`. The chain I'm going to work with is the following:

```swift
let eventHandler = { (event: Event<Int>) -> Void in
  print(event)
  }

Observable<Int>.interval(1, scheduler: MainScheduler.instance)
  .filter { $0 % 2 == 0 }
  .skip(2)
  .subscribe(eventHandler)
  .disposed(by: disposeBag)
```

I passed `eventHandler` closure to the `subscribe` method to emphasize what an observer is in our scenario. And if I added `.debug` after each operator, in the console  I would have the prints like this:

```swift
//interval -> Event next(0)
//filter -> Event next(0)
//interval -> Event next(1)
//interval -> Event next(2)
//filter -> Event next(2)
//interval -> Event next(3)
//interval -> Event next(4)
//filter -> Event next(4)
//skip -> Event next(4)
//interval -> Event next(5)
//interval -> Event next(6)
//filter -> Event next(6)
//skip -> Event next(6)
//interval -> Event next(7)
//interval -> Event next(8)
//filter -> Event next(8)
//skip -> Event next(8)
```

I looked through some files where these 3 operators are written. At first glance I got that for each operator there is not only Producer subclass, but also its counterpart represented by `Sink` class. The responsibility of Producer is simple - it creates Sink when `run` method invokes. Producers might have as many arguments as they need, and those arguments might be passed as initial arguments to Sink.

And for the operators above we'll check a pair of classes: 

* `Timer`+`TimerOneOffSink`;
* `Filter`+`FilterSink`;
* `SkipCount`+`SkipCountSink`;

OK, let's start with the pair `Timer`+`TimerOneOffSink`:

```swift
//Timer.swift
final private class Timer<E: RxAbstractInteger>: Producer<E> {
	override func run<O: ObserverType>(_ observer: O, cancel: Cancelable) -> (sink: Disposable, subscription: Disposable) where O.E == E {
		let sink = TimerOneOffSink(parent: self, observer: observer, cancel: cancel)
	    let subscription = sink.run()
	    return (sink: sink, subscription: subscription)
	}
}

final private class TimerOneOffSink<O: ObserverType>: Sink<O> where O.E: RxAbstractInteger {
    func run() -> Disposable {
        return self._parent._scheduler.scheduleRelative(self, dueTime: self._parent._dueTime) { [unowned self] _ -> Disposable in
            self.forwardOn(.next(0))
            self.forwardOn(.completed)
            self.dispose()

            return Disposables.create()
        }
    }
}

//DispatchQueueConfiguration.swift
func scheduleRelative<StateType>(_ state: StateType, dueTime: Foundation.TimeInterval, action: @escaping (StateType) -> Disposable) -> Disposable {
    let deadline = DispatchTime.now() + dispatchInterval(dueTime)

    let compositeDisposable = CompositeDisposable()

    let timer = DispatchSource.makeTimerSource(queue: self.queue)
    timer.schedule(deadline: deadline, leeway: self.leeway)

    timer.setEventHandler(handler: {
        if compositeDisposable.isDisposed {
            return
        }
        _ = compositeDisposable.insert(action(state))
        cancelTimer.dispose()
    })
    timer.resume()

    _ = compositeDisposable.insert(cancelTimer)

    return compositeDisposable
}
```

`Timer`-Producer creates `TimerOneOffSink` which creates `DispatchSourceTimer` for our repetitive job. The more you call `subscribe`, the more timers you will get. I mentioned it in [cold/hot post]({% post_url 2019-05-10-connectable-observable %}).

`Filter`+`FilterSink` pair is:

```swift
//Filter.swift
final private class Filter<Element>: Producer<Element> {
    override func run<O: ObserverType>(_ observer: O, cancel: Cancelable) -> (sink: Disposable, subscription: Disposable) where O.E == Element {
        let sink = FilterSink(predicate: self._predicate, observer: observer, cancel: cancel)
        let subscription = self._source.subscribe(sink)
        return (sink: sink, subscription: subscription)
    }
}

final private class FilterSink<O: ObserverType>: Sink<O>, ObserverType {   
    func on(_ event: Event<Element>) {
        switch event {
        case .next(let value):
            do {
                let satisfies = try self._predicate(value)
                if satisfies {
                    self.forwardOn(.next(value))
                }
            }
            catch let e {
                self.forwardOn(.error(e))
                self.dispose()
            }
        case .completed, .error:
            self.forwardOn(event)
            self.dispose()
        }
    }
}
```

Filter as Producer creates Sink when `run` method is called. `FilterSink` implementation doesn't have `run` method, this operator doesn't create anything, it forwards instead. `FilterSink` is `ObserverType` protocol, which means it implements `func on(_ event: Event<E>)` and it could be used in the subscription `self._source.subscribe(sink)`. Each time source receives any `Event` which is automatically forwarded to the subscribed `sink`. And Sink forwards Event further if the condition meets.

And finally, `SkipCount`+`SkipCountSink`:

```swift
//Skip.swift
final private class SkipCount<Element>: Producer<Element> {
    override func run<O : ObserverType>(_ observer: O, cancel: Cancelable) -> (sink: Disposable, subscription: Disposable) where O.E == Element {
        let sink = SkipCountSink(parent: self, observer: observer, cancel: cancel)
        let subscription = self.source.subscribe(sink)

        return (sink: sink, subscription: subscription)
    }
}

final private class SkipCountSink<O: ObserverType>: Sink<O>, ObserverType {
    func on(_ event: Event<Element>) {
        switch event {
        case .next(let value):
            
            if self.remaining <= 0 {
                self.forwardOn(.next(value))
            }
            else {
                self.remaining -= 1
            }
        case .error:
            self.forwardOn(event)
            self.dispose()
        case .completed:
            self.forwardOn(event)
            self.dispose()
        }
    }
}
```

`SkipCount` as Producer creates Sink when `run` method is called. `SkipCountSink` also works as a forwarder with some conditions in the implementation - `if self.remaining <= 0 { self.forwardOn(.next(value)) }`.

### Summarize

Ok, one more time let's take a look at our chain:

```swift
let eventHandler = { (event: Event<Int>) -> Void in
  print(event)
  }

Observable<Int>.interval(1, scheduler: MainScheduler.instance)	//3
  .filter { $0 % 2 == 0 }	//2
  .skip(2)	//1
  .subscribe(eventHandler)
  .disposed(by: disposeBag)
```

I'll try to write calls similarly as they are represented in stack. When you call `subscribe` after the last `skip` operator:

* creates an instance of `SinkDisposer`(1);
* calls `run` on `SkipCount`;
* sink(1) `SkipCountSink` is created;
* `self.source.subscribe(sink)` does the following: 
	* creates an instance of `SinkDisposer`(2);
	* calls `run` on `Filter`;
	* sink(2) `FilterSink` is created;
	* `self._source.subscribe(sink)` does the following:
		* creates an instance of `SinkDisposer`(3);
		* calls `run` on `Timer`;
		* sink(3) `TimerOneOffSink` is created;
		* call `run` on created sink;
		* subscription(3) is created using closure, where `DispatchSourceTimer` is created, which does the job for interval;
		* retains sink(3) and subscription(3) into `SinkDisposer`(3)
		* `SinkDisposer`(3) is returned;
	* `SinkDisposer`(3) is retained as subscription(2), also sink(2) is retained by `SinkDisposer`(2);
	* `SinkDisposer`(2) is returned;
* `SinkDisposer`(2) is retained as subscription(1), also sink(1) is retained by `SinkDisposer`(1);
* `SinkDisposer`(1) is returned;

`SinkDisposer`(1) is the instance which is added to `.disposed(by: disposeBag)`

Also each sink retains an observer as well:

* for sink(1) observer is anonymous closure `{ (event: Event<String>) -> Void in print(event) }`;
* for sink(2) observer is `skip` sink(1);
* for sink(3) observer is `filter` sink(2);

And let's check out the reverse process of calling `dispose()` on `SinkDisposer`(1):

* it disposes sink(1) and subscription(1)
* subscription(1) is `SinkDisposer`(2), it disposes sink(2) and subscription(2);
* subscription(2) is `SinkDisposer`(3), it disposes sink(3) and subscription(3);
* subscription(3) disposes source timer;

