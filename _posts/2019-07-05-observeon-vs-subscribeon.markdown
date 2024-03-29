---
title: "ObserveOn vs SubscribeOn"
date: 2019-07-05 12:00:00 +0300
categories: rxswift
toc: true
toc_sticky: true
---

By default, an Observable and the chain of operators that you apply to it will do its work, and will notify its observers, on the same thread on which its Subscribe method is called ([reactivex.io](http://reactivex.io/documentation/scheduler.html)).

That means, if you described an Observable and the chain of operators applied to it in `viewDidLoad` method without any dispatching, the subscription would be called on the Main thread as well as all closures for all operators would be executed on the Main thread.

Let's read the ReactiveX contract one more time:

By default, an Observable and the chain of operators that you apply to it will do its work, and **will notify its observers, on the same thread on which its Subscribe method is called**.

### Subscribe on Main

```swift
override func viewDidLoad() {
    super.viewDidLoad()
	
	Observable<Int>
	  .create { observer in
	    for i in 0..<20 {
	      sleep(1)
	      observer.onNext(i)
	    }
	    observer.onCompleted()
	    return Disposables.create()
	  }
	  .filter { num -> Bool in
	    return num % 2 == 0
	  }
	  .skipWhile { num -> Bool in
	    return num < 5
	  }
	  .map { num -> String in
	    return "current iteration is \(num)"
	  }
	  .bind(to: label.rx.text)
	  .disposed(by: disposeBag)
}
```

In the example above the subscribe method is called on the Main thread. If I put a breakpoint in each closure, I will see that all closures, which make creation and transformation, are running on the Main thread.

There is one interesting part in RxSwift implementation of the ReactiveX contract, all notification closures are called on the same thread where `observer.onNext(i)` is called. If I changed it to:

```swift
DispatchQueue.global().async {
	observer.onNext(i)
}
```

every notification closure would be called on the global queue.

### Subscribe on the default global queue

To subscribe on the default global queue without using handy rx operators I have to modify my code a bit:

```swift
  override func viewDidLoad() {
    super.viewDidLoad()

    let observable = Observable<Int>
      .create { observer in
        for i in 0..<20 {
          sleep(1)
          observer.onNext(i)
        }
        observer.onCompleted()
        return Disposables.create()
      }
      .filter { num -> Bool in
        return num % 2 == 0
      }
      .skipWhile { num -> Bool in
        return num < 5
      }
      .map { num -> String in
        return "current iteration is \(num)"
    }

    DispatchQueue.global().async {
      observable
        .bind(to: self.label.rx.text)
        .disposed(by: self.disposeBag)
    }
  }
```

Even the definition of the Observable was prepared on the Main thread, the subscribe method (`bind(to:)` in the example) changes this behavior and every notification closure is called on the global queue.

Let's try to break the ReactiveX contract one more time, and change `observer.onNext(i)` to:

```swift
DispatchQueue.main.async {
	observer.onNext(i)
}
```

Now every closure is called on the Main thread. 

The experiments above are not recommended to be used in real projects. There is a better rx-way to work with multithreading. The experiments above give the understanding how the chain of operators works with queues. As soon the N operator changes the queue in its closure, N+1's and every following operator's closures will be called on the changed queue, even though the subscribe method was called on a different one.

## observeOn operator 

This operator changes the scheduler (might be read as a queue in this context) for the notification closure of the operators which were applied **after** it.

An example of a notification closure for the `map` operator:

```swift
.map { num -> String in
	//closure
	return "current iteration is \(num)" 
}
```

### 1

Let's take a look at this pseudocode:

```swift
observable
	.operator1()
	.operator2()
	.operator3()
	.operator4()
	.observeOnBackground()
	.operator5()
	.operator6()
	.operator7()
	.operator8()
	.subscribe()
```

Assume we call `subscribe` method on the Main thread, if we hadn't added `observeOnBackground` in the middle of the chain, each operator's closure would have been called on the Main thread. But because we've added it, `observeOnBackground` explicitly instructs `operator5`-`operator8` and `subscribe` closures which queue they should use. Background in this case.

### 2

Let's take a look one more example:

```swift
observable
	.observeOnDefault()
	.operator1()
	.operator2()
	.operator3()
	.operator4()
	.observeOnBackground()
	.operator5()
	.operator6()
	.operator7()
	.observeOnMain()
	.operator8()
	.subscribe()
```

`observeOnDefault` instructs the right queue to all the following operators, but, after `operator4` there is a new instruction. And it cancels the previous one and instructs the new queue to all the following operators, in our case only for 3 operators - `operator5`-`operator7`. As you might have already guessed, `observeOnMain` cancels the previous instruction from `observeOnBackground` and `operator8` and `subscribe` closure will be called on the Main thread.

### 3

And the final example:

```swift
observable
	.operator1()
	.operator2()
	.operator3()
	.operator4()
	.operator5()
	.operator6()
	.operator7()
	.operator8()
	.observeOnBackground()
	.observeOnDefault()
	.observeOnMain()
	.subscribe()
```

On what queue will the closure of the final `subscribe` be called? The right answer - on the Main thread. `observeOnMain ` cancels the previous instruction which was `observeOnDefault`. It doesn't matter how many `observeOn` operators in the line are, cause the closest preceding operator cancels the previous instructions.

## subscribeOn operator

This operator changes the scheduler (might be read as a queue in this context) for subscription hassles for all operators which were applied **before** it. By subscription hassles I mean the `run` method and everything related to the process of creating a particular instance of the sink-part of some operator.

If you need more details about [subscription topic]({% post_url 2019-05-31-what-subscribe-does %}) you might want to check the article where I described how the `Producer` subclasses and the `Sink` help separating responsibilities in RxSwift framework.

A `Producer` instance knows what it needs to create, a `Sink` instance knows the logic of a particular operator. When subscribe is called, usually the `run` method is called on a producer's instance. `subscribeOn` instructs producers on what queue they have to call the `run` method. For creating operators, it points to the queue where the initial closure should be called.

```swift
 override func viewDidLoad() {
    super.viewDidLoad()

    Observable<Int>
      .create { observer in
        for i in 0..<20 {
          sleep(1)
          observer.onNext(i)
        }
        observer.onCompleted()
        return Disposables.create()
      }
      .filter { num -> Bool in
        return num % 2 == 0
      }
      .skipWhile { num -> Bool in
        return num < 5
      }
      .map { num -> String in
        return "current iteration is \(num)"
      }
      .subscribeOn(ConcurrentDispatchQueueScheduler(qos: .default))
      .bind(to: label.rx.text)
      .disposed(by: self.disposeBag)
}
```

In the example above `subscribeOn` instructs Map Producer to change the default queue while calling the `run` method, which contains Sink creation:

```swift
let sink = MapSink(transform: self._transform, observer: observer, cancel: cancel)
let subscription = self._source.subscribe(sink)
return (sink: sink, subscription: subscription)
```

`subscribeOn` affects not only one `run` call, it instructs all preceding producers to call their `run` method on a particular queue.

### 1 

Let's jump back to the pseudocode:

```swift
observable
	.operator1()
	.operator2()
	.operator3()
	.operator4()
	.subscribeOnBackground()
	.operator5()
	.operator6()
	.operator7()
	.operator8()
	.subscribe()
```

In this case `subscribeOnBackground` instructs `operator1`-`operator4` and `observable` to change the queue when the `run` method is called.

### 2

```swift
observable
	.subscribeOnBackground()
	.operator1()
	.operator2()
	.operator3()
	.operator4()
	.subscribeOnGlobal()
	.operator5()
	.operator6()
	.operator7()
	.operator8()
	.subscribe()
```

In this case only the `run` method of  `observable` will be called on the `Background`. For `operator1`-`operator4` the `run` method will be called on the global queue. For `operator5`-`operator8` the `run` method will be called on the same queue where this piece of code is written, if it had been in the `viewDidLoad` method - the Main queue would have been used.

### 3

```swift
observable
	.subscribeOnBackground()
	.subscribeOnDefault()
	.subscribeOnMain()
	.operator1()
	.operator2()
	.operator3()
	.operator4()
	.operator5()
	.operator6()
	.operator7()
	.operator8()
	.subscribe()
```

On what queue will the initial `observable` closure be called? The right answer - on the Background thread. Similarly to the `observeOn`, but in this case we are looking at the first `subscribeOn` entry.

### ObserveOn vs SubscribeOn

* `observeOn` instructs the following operators;
* `subscribeOn` instructs the preceding operators;
* `observeOn` affects Sink's `func on(_:)` method;
* `subscribeOn` affects Producer's `func run(_:cancel:)` method;

I like the way how Marin [described this topic](http://rx-marin.com/post/observeon-vs-subscribeon/) in his blog.
