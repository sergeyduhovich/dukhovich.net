---
title: "Delegates"
date: 2019-07-26 12:00:00 +0300
categories: rxswift
toc: true
toc_sticky: true
---

Operators from create section are not the only way to create an Observable sequence.

From time to time you might want to add some reactiveness to the code based on [delegate pattern](https://developer.apple.com/documentation/swift/cocoa_design_patterns/using_delegates_to_customize_object_behavior). This pattern is commonly used in Apple's frameworks, especially in UIKit. 

### UITableView

Let's take a look at most used class with delegate/datasource pattern. It's a `UITableView`.

The simplest RxSwift `UITableView` example might look like the following:

```swift
class ViewController: UIViewController {

  @IBOutlet var tableView: UITableView!

  private let disposeBag = DisposeBag()
  private let mostVisitedCities = Observable<[String]>.just([
    "Bangkok",
    "London",
    "Paris",
    "Dubai",
    "Singapore",
    "New York",
    "Kuala Lumpur",
    "Tokyo",
    "Istanbul",
    "Seoul",
    "Antalya",
    "Phuket",
    "Mecca",
    "Hong Kong",
    "Milan",
    "Palma de Mallorca",
    "Barcelona",
    "Pattaya",
    "Osaka",
    "Bali"
    ])

  override func viewDidLoad() {
    super.viewDidLoad()
    
    tableView.register(UITableViewCell.self, forCellReuseIdentifier: "cell")
    mostVisitedCities
      .bind(to: tableView.rx.items(cellIdentifier: "cell", cellType: UITableViewCell.self)) { index, model, cell in
        cell.textLabel?.text = model
      }
      .disposed(by: disposeBag)
  }
}
```

![countries](http://dukhovich.by/assets/images/articles/countries.png)

When you connect table's outlet and the controller in storyboard/xib together (without setting a dataSource property), you'll see that the list above is displayed in `UITableView` properly. You might have noticed, that I hadn't setup dataSource in code as well. 

The dataSource <-> viewcontroller connection was done in `items(dataSource:)` call. You could check the implementation out opening `UITableView+Rx.swift`. There is an interesting entry of `RxTableViewDataSourceProxy` which will be my start point for  this topic.

Without body, the class definition and inheritance chain look in the following way:

```swift
open class RxTableViewDataSourceProxy
    : DelegateProxy<UITableView, UITableViewDataSource>
    , DelegateProxyType 
    , UITableViewDataSource {
}
```

We see `UITableViewDataSource` protocol, class `DelegateProxy` with specific types for this narrow use-case and `DelegateProxyType` protocol.

Looking into `DelegateProxyType.swift` and reading its short documentation explains  the idea behind the class mess above. As developers, we might want to use both, regular delegate calls and reactive wrappers for them at the same time (what might be not the best idea).

There is a warning about usage:

> Type implementing `DelegateProxyType` should never be initialized directly.
> To fetch initialized instance of type implementing `DelegateProxyType`, `proxy` method should be used.

For some classes, like UITableView, initialization is already done. All we need is just write `tableView.rx.setDelegate(self)` in our ViewController.

### "Broken" delegate

Let's do a short test for standard delegate. I've added `didSelectRowAt` method to controller's extension:

```swift
extension ViewController: UITableViewDelegate {
  func tableView(_ tableView: UITableView, didSelectRowAt indexPath: IndexPath) {
    print("call from extension")
  }
}
```

also, I've added in `viewDidLoad` the following code:

```swift
tableView.rx.itemSelected
  .subscribe(onNext: { indexPath in
    print("call from rx.itemSelected")
  })
  .disposed(by: disposeBag)
```

The most interesting part for standard `delegate` property for `UITableView`:

If I put `tableView.delegate = self` **before** `tableView.rx.itemSelected` both prints will be called;

```swift
//call from extension
//call from rx.itemSelected
```

If I put `tableView.delegate = self` **after** `tableView.rx.itemSelected` only print from extension(standard) will be called and reactive version `tableView.rx.itemSelected` will be ignored;

```swift
//call from extension
```

To avoid ambiguity and provide predictable behavior for RxSwift usage, all those proxies in RxCocoa has been written.

```swift
tableView.rx.setDelegate(self)
  .disposed(by: disposeBag)
```

And the code above is not sensitive for the place where it's written, before or after other pieces of rx-code.

### UINavigationController

RxCocoa has a lot of UIKit extensions. Let's test how `UINavigationController` delegate extension works. RxCocoa provides 2 ControlEvent: `willShow` and `didShow` the same as `UINavigationControllerDelegate` does.

At first, I conformed `UINavigationControllerDelegate`:

```swift
extension ViewController: UINavigationControllerDelegate {
  func navigationController(_ navigationController: UINavigationController, willShow viewController: UIViewController, animated: Bool) {
    print("call from extension")
  }
}
```

Also, I added the subscription to `willShow` ControlEvent:

```swift
navigationController?.rx.willShow
  .subscribe(onNext: { (controller, animated) in
    print("call from rx.willShow")
  })
  .disposed(by: disposeBag)
```

After that, I should setup the delegate `navigationController?.delegate = self`. And as for the UITableView example, there is the same unobvious behavior. If I setup delegate **before** subscription to `willShow` ControlEvent, everything works as expected:

```swift
//call from extension
//call from rx.willShow
```

If I setup delegate **after** subscription to `willShow` ControlEvent, only regular extension method is triggered:
 
```swift
//call from extension
```

The rescue here is `RxNavigationControllerDelegateProxy` class. When you setup the following lines in the `viewDidLoad` method, ambiguity will gone.

```swift
RxNavigationControllerDelegateProxy
  .installForwardDelegate(self,
                          retainDelegate: false,
                          onProxyForObject: navigationController!)
  .disposed(by: disposeBag)
```

But, I recommend using extension if it's really needed. And for the last example we could rewrite a bit our implementation:

```swift
extension Reactive where Base: UINavigationController {
  func setDelegate(_ delegate: UINavigationControllerDelegate)
    -> Disposable {
      return RxNavigationControllerDelegateProxy
        .installForwardDelegate(delegate,
                                retainDelegate: false,
                                onProxyForObject: self.base)
  }
}
```

and `viewDidLoad` method will look much nicer:

```swift
navigationController?.rx
  .setDelegate(self)
  .disposed(by: disposeBag)
```

### Custom delegate

We've covered a bit of extensions written in RxCocoa. It's time to write our own rx-delegate. As an example, I'll show you how to write rx-wrapper for `CNContactPickerDelegate`.

There are parts of custom delegate implementation. At first, `RxCNContactPickerDelegateProxy`:

```swift
/// For more information take a look at `DelegateProxyType`.
open class RxCNContactPickerDelegateProxy
  : DelegateProxy<CNContactPickerViewController, CNContactPickerDelegate>
  , DelegateProxyType
, CNContactPickerDelegate {

  /// Typed parent object.
  public weak private(set) var pickerController: CNContactPickerViewController?

  /// - parameter navigationController: Parent object for delegate proxy.
  public init(pickerController: ParentObject) {
    self.pickerController = pickerController
    super.init(parentObject: pickerController, delegateProxy: RxCNContactPickerDelegateProxy.self)
  }

  // Register known implementations
  public static func registerKnownImplementations() {
    self.register { RxCNContactPickerDelegateProxy(pickerController: $0) }
  }
}
```

As for me, the easiest way to write implementation above is modifying existed proxy  (`RxNavigationControllerDelegateProxy` as an example) by your needs, replacing entries `UINavigationController` by `CNContactPickerViewController` and `UINavigationControllerDelegate` by `CNContactPickerDelegate`.

Next part is `Reactive` extension. Here we need at least a `delegate` property. Other methods are optional. But we aim to implement everything from delegate's method list:

```swift
extension Reactive where Base: CNContactPickerViewController {

  /// Reactive wrapper for `delegate`.
  ///
  /// For more information take a look at `DelegateProxyType` protocol documentation.
  public var delegate: DelegateProxy<CNContactPickerViewController, CNContactPickerDelegate> {
    return RxCNContactPickerDelegateProxy.proxy(for: base)
  }

  /// Reactive wrapper for delegate method `contactPicker(_:didSelect:)`.
  var didSelectContact: ControlEvent<CNContact> {
    let sel = #selector((CNContactPickerDelegate.contactPicker(_:didSelect:)! as (CNContactPickerDelegate) ->  (CNContactPickerViewController, CNContact) -> Void))
    let source: Observable<CNContact> = delegate.methodInvoked(sel)
      .map { arg in
        let contact = arg[1] as! CNContact
        return contact
    }
    return ControlEvent(events: source)
  }

  /// Reactive wrapper for delegate method `contactPicker(_:didSelect:)`.
  var didSelectContactProperty: ControlEvent<CNContactProperty> {
    let sel = #selector((CNContactPickerDelegate.contactPicker(_:didSelect:)! as (CNContactPickerDelegate) ->  (CNContactPickerViewController, CNContactProperty) -> Void))
    let source: Observable<CNContactProperty> = delegate.methodInvoked(sel)
      .map { arg in
        let contact = arg[1] as! CNContactProperty
        return contact
    }
    return ControlEvent(events: source)
  }
}
```

The most painful in this implementation were these selectors:

```swift
let sel = #selector((CNContactPickerDelegate.contactPicker(_:didSelect:)! as (CNContactPickerDelegate) -> (CNContactPickerViewController, CNContact) -> Void))

let sel = #selector((CNContactPickerDelegate.contactPicker(_:didSelect:)! as (CNContactPickerDelegate) -> (CNContactPickerViewController, CNContactProperty) -> Void))
```

Because, from objc point of view, they are absolutely the same, and the error appears `Ambiguous use of 'contactPicker(_:didSelect:)'`. To solve the ambiguity, we must add type castings:

```swift
(CNContactPickerDelegate) -> (CNContactPickerViewController, CNContact) -> Void)
(CNContactPickerDelegate) -> (CNContactPickerViewController, CNContactProperty) -> Void)
``` 

Also, I don't like force type casting in `map` closure, but it's the way how `methodInvoked(_:)` is implemented in `DelegateProxy`.

```swift
.map { arg in
        let contact = arg[1] as! CNContactProperty
        return contact
    }
```

And the final extension for delegate:

```swift
extension Reactive where Base: CNContactPickerViewController {
  func setDelegate(_ delegate: CNContactPickerDelegate)
    -> Disposable {
      return RxCNContactPickerDelegateProxy
        .installForwardDelegate(delegate,
                                retainDelegate: false,
                                onProxyForObject: self.base)
  }
}
```

We've already seen similar implementations above.

Source code for this article is available on [github](https://github.com/SergeyDukhovich/DelegateRxSample).