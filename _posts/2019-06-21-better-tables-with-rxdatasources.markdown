---
title: "Better tables with RxDataSources"
date: 2019-06-21 12:00:00 +0300
categories: rxswift
toc: true
toc_sticky: true
---

RxCocoa framework provides some methods that help you bind an Observable sequence to an instance of UITableView or UICollectionView. 

## RxCocoa

The task for this article is to display an array of `Message` in UITableView.

```swift
enum Message {
  case text(String)
  case attributedText(NSAttributedString)
  case photo(UIImage)
  case location(lat: Float, lon: Float)
}
```

For the `text` and the `attributedText` cases I'm going to use standard cells. And for the `photo` and the `location` cases I'm going to create custom xibs. My message source will be the following:

```swift
[
    .attributedText(NSAttributedString(string: "Blue text",
                                       attributes: [NSAttributedString.Key.foregroundColor : UIColor.blue])),
    .text("On the other hand, we denounce with righteous indignation and dislike men who are so beguiled and demoralized by the charms of pleasure of the moment"),
    .photo(UIImage(named: "rx-wide")!),
    .attributedText(NSAttributedString(string: "Red text",
                                       attributes: [NSAttributedString.Key.foregroundColor : UIColor.red])),
    .text("Another Message"),
    .location(lat: 37.334722, lon: -122.008889),
    .text("Lorem Ipsum is simply dummy text of the printing and typesetting industry. Lorem Ipsum has been the industry's standard dummy text ever since the 1500s"),
    .photo(UIImage(named: "rx-logo")!),
    .attributedText(NSAttributedString(string: "Green text",
                                       attributes: [NSAttributedString.Key.foregroundColor : UIColor.green])),
    .location(lat: 53.9, lon: 27.56667),
    .text("There are many variations of passages of Lorem Ipsum available, but the majority have suffered alteration in some form, by injected humour, or randomised words which don't look even slightly believable. If you are going to use a passage of Lorem Ipsum, you need to be sure there isn't anything embarrassing hidden in the middle of text."),
    .attributedText(NSAttributedString(string: "Yellow text",
                                       attributes: [NSAttributedString.Key.foregroundColor : UIColor.yellow])),
]
```

### Bind source to table

RxCocoa provides 2 methods for binding a source array to UITableView. The examples from the `UITableView+Rx.swift` are the following:

#### 1.

```swift
 let items = Observable.just([
     "First Item",
     "Second Item",
     "Third Item"
 ])

 items
     .bind(to: tableView.rx.items(cellIdentifier: "Cell", cellType: UITableViewCell.self)) { (row, element, cell) in
        cell.textLabel?.text = "\(element) @ row \(row)"
     }
     .disposed(by: disposeBag)
```

This method will be suitable when you are going to use just one cell type for all items from the source. The cell identifier and cell type can't be changed in the closure. All you need to do in the closure is to configure the existing cell before it is displayed.

#### 2.

```swift
 let items = Observable.just([
     "First Item",
     "Second Item",
     "Third Item"
 ])

 items
 .bind(to: tableView.rx.items) { (tableView, row, element) in
     let cell = tableView.dequeueReusableCell(withIdentifier: "Cell")!
     cell.textLabel?.text = "\(element) @ row \(row)"
     return cell
 }
 .disposed(by: disposeBag)
```

This method gives you more control over the cell types. Now you are responsible for the cell creation process as well.

Using this method I can implement a table displaying the data source I wanted. All these static class functions just update the cell using a message.

```swift
messages
  .bind(to: tableView.rx.items) { (table: UITableView, index: Int, message: Message) in
    guard let cell = table.dequeueReusableCell(withIdentifier: message.identifier.rawValue) else { return UITableViewCell() }
    switch message {
    case let .text(message):
      return ViewController.configure(text: message, cell: cell)
    case let .attributedText(attributed):
      return ViewController.configure(attributed: attributed, cell: cell)
    case let .photo(photo):
      return ViewController.configure(photo: photo, cell: cell)
    case let .location(lat: lat, lon: lon):
      return ViewController.configure(lat: lat, lon: lon, cell: cell)
    }
  }
.disposed(by: disposeBag)
```

The table displaying all messages in different cell types:

![rxcocoa-table-sample](http://dukhovich.by/assets/images/articles/10/rxcocoa-table-sample.png)

[Link to the commit with current implementation.](https://github.com/SergeyDukhovich/RxDataSourcesSample/tree/326ca6e6a182d54a184edbec1a1fd9bcf288a733)

And if RxCocoa table binding works fine, why is RxDataSources needed?

## RxDataSources

### Sections

RxDataSources allows us to bind not just a single sectioned data source, but multiple as well. Let's say I want to split 12 messages into 3 sections, 4 messages in each.

The framework provides 2 types of data sources for tables: `RxTableViewSectionedReloadDataSource` and `RxTableViewSectionedAnimatedDataSource`, and similar types for collections. These classes require implementation of `SectionModelType` protocol. Hopefully,  `SectionModel` struct from internal dependency (Differentiator) could be used for this purpose.  

And our initial source Observable with this type: `Observable<[Message]>` will be modified to the `Observable<[SectionModel<String, Message>]>`. 

There is a new property in ViewController, which is called `dataSource`:

```swift
 let dataSource = RxTableViewSectionedReloadDataSource<SectionModel<String, Message>>(configureCell: { dataSource, table, indexPath, message in
    guard let cell = table.dequeueReusableCell(withIdentifier: message.identifier.rawValue) else { return UITableViewCell() }
    switch message {
    case let .text(message):
      return ViewController.configure(text: message, cell: cell)
    case let .attributedText(attributed):
      return ViewController.configure(attributed: attributed, cell: cell)
    case let .photo(photo):
      return ViewController.configure(photo: photo, cell: cell)
    case let .location(lat: lat, lon: lon):
      return ViewController.configure(lat: lat, lon: lon, cell: cell)
    }
  })
```

Also, `messages` variable was updated: 

```swift
private var messages = Observable<[SectionModel<String, Message>]>.just([
    SectionModel<String, Message>(model: "1",
                                  items: [
                                    .attributedText(NSAttributedString(string: "Blue text",
                                                                       attributes: [NSAttributedString.Key.foregroundColor : UIColor.blue])),
                                    .text("On the other hand, we denounce with righteous indignation and dislike men who are so beguiled and demoralized by the charms of pleasure of the moment"),
                                    .photo(UIImage(named: "rx-wide")!),
                                    .attributedText(NSAttributedString(string: "Red text",
                                                                       attributes: [NSAttributedString.Key.foregroundColor : UIColor.red]))
      ]),
    SectionModel<String, Message>(model: "2",
                                  items: [
                                    .text("Another Message"),
                                    .location(lat: 37.334722, lon: -122.008889),
                                    .text("Lorem Ipsum is simply dummy text of the printing and typesetting industry. Lorem Ipsum has been the industry's standard dummy text ever since the 1500s"),
                                    .photo(UIImage(named: "rx-logo")!),
                                    .attributedText(NSAttributedString(string: "Green text",
                                                                       attributes: [NSAttributedString.Key.foregroundColor : UIColor.green]))
      ]),
    SectionModel<String, Message>(model: "3",
                                  items: [
                                    .location(lat: 53.9, lon: 27.56667),
                                    .text("There are many variations of passages of Lorem Ipsum available, but the majority have suffered alteration in some form, by injected humour, or randomised words which don't look even slightly believable. If you are going to use a passage of Lorem Ipsum, you need to be sure there isn't anything embarrassing hidden in the middle of text."),
                                    .attributedText(NSAttributedString(string: "Yellow text",
                                                                       attributes: [NSAttributedString.Key.foregroundColor : UIColor.yellow]))
      ])
    ])
```

With the changes above our table will look absolutely the same. To modify the appearance a bit, let's add section titles. This code should be added before the line where the actual binding is.

```swift
    dataSource.titleForHeaderInSection = { dataSource, index in
      return dataSource.sectionModels[index].model
    }
```

The table displaying all messages in different cell types in 3 sections:

![rxdatasources-table-sample](http://dukhovich.by/assets/images/articles/10/rxdatasources-table-sample.png)

[Link to the commit with current implementation.](https://github.com/SergeyDukhovich/RxDataSourcesSample/tree/c3c55b3a7e484d2b067f182ffee492f99e22a67b)

### Animation 

The implementation above uses `reloadData` method on the UITableView when observable changes. RxDataSources also provides the animated way of updating tables.

At first, all our messages should conform to `IdentifiableType` protocol. At first glance it might seem like we need to conform our enum in the similar way as the following implementation:

```swift
extension Message: IdentifiableType {
  var identity : String {
    switch self {
    case let .text(text):
      return "text_\(text)"
    case let .attributedText(text):
      return "attributed_\(text)"
    case let .location(lat: lat, lon: lon):
      return "\(lat)_\(lon)"
    case let .photo(image):
      guard let data = image.pngData() else { return "image" }
      return String(data.hashValue)
    }
  }
}
```

But don't be in a hurry. In some cases, like in a chat implementation, you might want to have unique identifiers for the cells even with the same text. Like "Hello"-cell from a user John and "Hello"-cell from a user Adam should be different. There are no compile-time errors in the implementation above, but it doesn't suit the chat.

Ok. The bottom line- what do we have to change in our project?

* Change `RxTableViewSectionedReloadDataSource` to `RxTableViewSectionedAnimatedDataSource`;
* Change `SectionModel` to `AnimatableSectionModel`;
* Conform `Message` to `IdentifiableType` protocol;
* Conform `Message` to `Equatable` protocol;

[Link to the commit with current implementation.](https://github.com/SergeyDukhovich/RxDataSourcesSample/tree/1cd8dd53296739df4871978dea7bd54a2be3840a)

### NSObject

From time to time we need to conform a subclass of `NSObject` to `IdentifiableType`.

In the last example I'll provide a table for the `MessageObject`:

```swift
class MessageObject: NSObject {
  let message: Message
  let messageId: String

  init(message: Message, messageId: String) {
    self.message = message
    self.messageId = messageId
  }

  override func isEqual(_ object: Any?) -> Bool {
    guard let obj = object as? MessageObject else { return false }
    return messageId == obj.messageId
  }
}
```

`Message` is the same as we used in the examples above.

![rxdatasources-table-sample](http://dukhovich.by/assets/images/articles/10/rxdatasources-animated-implementation.gif)

[Link to the commit with current implementation.](https://github.com/SergeyDukhovich/RxDataSourcesSample/tree/115397f2bbc86986e7454a01a00e45999228b807)
