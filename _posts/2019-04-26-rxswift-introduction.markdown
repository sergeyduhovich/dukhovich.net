---
title: "Reactive programming"
date: 2019-04-26 12:00:00 +0300
categories: rxswift
toc: true
toc_sticky: true
---

In computing reactive programming is a declarative programming paradigm concerned with data streams and the propagation of change. You might want to read more about reactive programming on the [Wikipedia](https://www.wikiwand.com/en/Reactive_programming), but there is also a good post on [stackoverflow](https://stackoverflow.com/a/1030631) 

In this post I’m going to introduce you a reactive framework called RxSwift, which is known as a port of [ReactiveX](http://reactivex.io/) to Swift.

> ReactiveX is a combination of the best ideas from the Observer pattern, the Iterator pattern and functional programming

ReactiveX is based on Observer and Iterator patterns. They are the ground of RxSwift and you might have not only single but also multiple Observers listening to a single Observable. Other [parts of their docs](http://reactivex.io/documentation/observable.html) say that it's based on Reactor and Iterator patterns. Don't worry about it, we don't need to know the difference between the observer and reactor, in terms of ReactiveX it’s ok to use one of the definitions, let’s say Observer. 

## An example

There is an example of RxSwift usage taken from their github repository:

At first you need to define Observable `searchResults` for GitHub repositories:

```swift
let searchResults = searchBar.rx.text.orEmpty
  .throttle(0.3, scheduler: MainScheduler.instance)
  .distinctUntilChanged()
  .flatMapLatest { query -> Observable<[Repository]> in
    if query.isEmpty {
      return .just([])
    }
    return searchGitHub(query)
      .catchErrorJustReturn([])
  }
  .observeOn(MainScheduler.instance)
```

Then you need to bind the results to your UITableView:

```swift
searchResults
  .bind(to: tableView.rx.items(cellIdentifier: "Cell")) {
    (index, repository: Repository, cell) in
    cell.textLabel?.text = repository.name
    cell.detailTextLabel?.text = repository.url
  }
  .disposed(by: disposeBag)
```

As a result, you'll see how it works:

![tableview](http://dukhovich.by/assets/images/articles/GithubSearch.gif)


## Pros of RxSwift

In this post I'll try to gather pros and cons and share my opinion about them.

And before I start writing pros, let's take a look at the following examples:

```swift
_ = Observable<String>
  .just("hello world") //2
  .subscribe(onNext: { element in //3
    print(element) //1
  })
```

```swift
observable //2
  .map { element in //4
    return "current element is \(element)"
  }
  .observeOn(ConcurrentDispatchQueueScheduler(qos: .background)) //4
  .subscribe(onNext: { element in //3
    print(element) //1
  })
  .disposed(by: disposeBag) //5
```

### 1. Compact and linear writing form

The biggest bonus of using Rx is its writing style.

The examples above are written in the similar way:

1. We always have code which is doing some helpful work (don’t judge me for the print, it is also a piece of work);
1. There is an observable stream;
1. There is a subscribe call to attach the observable stream to a particular observer (in the examples above our observer is just an unnamed closure);
1. There might be or might not be an additional transformation block or blocks;
1. And finally, if we are not interested in memory leaks, we usually add our chain to some sort of memory manager implementation, which is known as Disposable.

If you have  experience in iOS development, you might be familiar with all these dissimilar API and techniques, such as the data-source and the delegate pattern for tables, the target-action pattern for button clicks, subscriptions to the Notification Center and key-value-observing. All implementations of these patterns are varied and your file, where you try to implement some logic, might end up with being fragmented and hardly ever maintained. And that’s the place where Rx comes with its writing style.

I personally like the statement “write once - read many times”, which I've heard from Scott Gardner. It works for RxSwift chains. To be honest, if the initial task was modified a lot, you might find that it is much easier to rewrite the whole chain, rather than modify it.

Let’s take a look at a few more examples of RxSwift usage in Project:

```swift
struct Hotel {
  let title: String
  let rating: Float
}

let hotelsObservable: Observable<[Hotel]> = ...

hotelsObservable
  .observeOn(MainScheduler.instance)
  .bind(to: tableView.rx.items(cellIdentifier: "cell", cellType: UITableViewCell.self)) { index, model, cell in
    cell.textLabel?.text = model.title
}
.disposed(by: disposeBag)

hotelsObservable
  .observeOn(MainScheduler.instance)
  .map { "we found \($0.count) in your range" }
  .bind(to: label.rx.text)
  .disposed(by: disposeBag)

tableView.rx.modelSelected(Hotel.self)
  .subscribe(onNext: { [weak self] hotel in
    self?.openDetails(hotel: hotel)
  })
  .disposed(by: disposeBag)

NotificationCenter.default.rx
  .notification(UIApplication.didEnterBackgroundNotification)
  .subscribe(onNext: { [weak self] _ in
    self?.saveData()
  })
  .disposed(by: disposeBag)

button.rx
  .tap
  .bind { [weak self] in
    self?.refreshData()
  }
  .disposed(by: disposeBag)
```

### 2. Crossplatform

RxSwift is the Swift version of ReactiveX, all knowledge you gain will be portable and you’ll be able to apply it in other programming languages or at least discuss it with your colleague programming in a different language.

### 3. Easy to use threads

Switching between threads in RxSwift is extremly easy. All you need is just to add one extra operation - `.observeOn(...)` or `.subscribeOn(...)`.

### 4. Community

Another big plus of RxSwift is its community. Enormous extensions are waiting for you, just take them. RxCocoa for UI, RxDataSources for tables, RxAlamofire for networking, RxCoreLocation for location service and many others.

### 5. Suits for MVVM-like

RxSwift suits MVVM-like architectures not only because there is a simple way to bind View to ViewModel. It is also easy to use operators to transform Observable signals from children ViewModels and create a new Observable for a parent ViewModel. And to be honest, you don't need to write a lot of code. At the same time if you need more control for ViewModels, you might find `BehaviourRelay`s and `PublishRelay`s useful.

## Cons of RxSwift

### 1. Overhead

And here comes the question: if RxSwift is so remarkable and helpful, why is it not a part of the Foundation framework? It is because with usability and maintainability you add a bit of overhead to your project. You have to follow some architectural idioms added on top of Apple's ones.


### 2. Learning curve

It might be difficult to get started with RxSwift. It may take 1 or 2 weeks to get true understanding what is going on. But as I said, there is a good community and there are a lot of examples in the github repository. 

### 3. Complex debug

Especially at the beginning, you might find it difficult to understand what is happening after minor changes, or when some error appears. 
The stack trace is very big. Without extra prints it's hard to get that your Observable was completed when you are not expecting it, especially due to an error.


## To add or not to add?

Let’s analyse the question above. I’ve got some explanations for both “Yes” and “No” answers.

### Yes

RxSwift helps you to implement a complex chain of operations. A chain with any complexity might be implemented in the same way as we described above in those 5 steps. There is an enormous number of working just fine transforming operators which are waiting for your attention. The framework contains Subjects which may be a bridge between the imperative and declarative world. Subjects work as an observable on the one hand and as Observer on the other. You might easily pass a new value to it to trigger its Observable. Besides, RxSwift works with MVVM-like architectures in a good way as well.

### No

You don’t have any experience and you don’t have any mentor on the project you are currently working on, and you are considering to start using RxSwift in this project, even worse, the deadline is coming. In this case you should avoid using RxSwift on the project, (здесь нужно какое-то из условий, если вы не ..., то лучше избегать rxswift)
as it's not a panacea, it's just one of possible ways to write applications.

## Event

> ... data streams and the propagation of change over time

In terms of RxSwift what is meant by data is what is known as `enum Event` with three possible cases: `next`, associated with generic value; `error`, associated with Error and `completed`.


```swift
public enum Event<Element> {
	/// Next element is produced.
	case next(Element)
	
	/// Sequence terminated with an error.
	case error(Swift.Error)
	
	/// Sequence completed successfully.
	case completed
}
```

Each Observable might emit zero or more elements. In most cases events will be emitted only if there is at least one Observer listening to Observable. (It is known as a cold observable, I’m going to cover this part in the future posts). There is no limitation for types to use them in Event, but there is one rule: one Observable can’t emit two or more different types, let’s say `Int` and `String`. Of course, it is possible to have Observable working with one single protocol type and two different types which are conformed to this protocol, but it is a very rare case.

Let's take a look at the Marble diagram. You’ll see a lot of [them](https://rxmarbles.com/) when you start working with RxSwift. I find it useful for a learning purpose.

![rxmarbles_debounce](http://dukhovich.by/assets/images/articles/post_1_rx_marbles.png)

Circles on the diagram are the following events: the vertical line is the completed event and the cross sign is the error event. **Normal termination** of an Observable sequence is the Observable with **Completed** or **Error** event. After one of them is emitted, Observable can’t emit any Next event and resources for Observable are released.

It’s the ground of RxSwift, the earlier you get this concept, the better your code will be and the better understanding you will have. When **Error** or **Completed** events are emitted, the whole **Observable** sequence will be **terminated**. That’s it, no future Next events. I will cover the techniques in my future posts and we will discuss how to handle errors and make Observable alive longer than it could.

Let’s take a glance one more time at the code block above with `enum Event`. Why Error is not a generic type but abstract instead. There is a [good explanation](https://github.com/ReactiveX/RxSwift/blob/master/Documentation/DesignRationale.md) for this purpose.
In general, the simplest an error is , the easier it is to combine observables.

## What DisposeBag and Disposable concept are?

RxSwift has an additional memory tool to help deal with ARC and memory management. DisposeBag is a container for subscriptions which are released when its parent object (class with `disposeBag: DisposeBag` property) is deallocated. 

Without DisposeBag you’ll have a retain cycle of both Observable and Observer. It is nice to follow a good practice and add every subscription to the disposeBag (there are a few exceptions, such as `.just()`, `.empty()`, but it is not a mistake to use DisposeBag for any subscription you create).

Each time you release (or reassign) DisposeBag you release resources allocated for calculations. And you terminate all Observable sequences as well.

## Cold / Hot Observables

Another fundamental thing of RxSwift is [understanding cold/hot Observables]({% post_url 2019-05-10-connectable-observable %}). By design, all Observable sequences created with operators from the [create](http://reactivex.io/documentation/operators/create.html) section are cold. What does it mean? Each time you call `subscribe` on cold signal, one more Observable is created and resources are allocated for calculations. In some cases, you might want to have “shared” observable variable in small scope where there is only one Observable and a few transformations on this observable applies. And before you call `subscribe` nothing happens, your chain is just waiting for your call `subscribe`.

Hot observable is the opposite. It emits events even if there is not any subscriber. It might be a socket or a pushing value, it might be text property of `UITextField` object, also it might be connectable Observable. Hot observable behaves in the same way no matter if there are 0 subscribers or there are dozen of them.
