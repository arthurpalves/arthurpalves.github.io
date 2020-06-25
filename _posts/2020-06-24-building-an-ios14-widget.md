---
layout:     post
title:      Building an iOS 14 Widget to Show Account Balance
date:       2020-06-24
summary:    iOS 14 has introduced Widgets. Here we create our very first widget to show balance and details of a bank account.
tags: 		ios14 swiftlang development swift ios widget
categories: tutorial
featured_image: /images/posts/04_ios_widget.png
---

> **Disclaimer:** This is merely used as proof of concept rather than a production feature, as there are security points to be taken in consideration for showing private and sensitive data for banking apps and similar applications.

[WWDC20](https://developer.apple.com/wwdc20/) has introduced us to [iOS 14](https://www.apple.com/ios/ios-14-preview/), which along other features now include widgets on the home screen. Widgets aren't new, but now they are totally different - including the way you build them, support up to 3 different sizes and **are on the home screen! 😱**

## Why is this a big deal for us?

Right there on your home screen you have meaningful information from your apps without even opening them. It's meant to save you time and extend your experience. It's an extension of your application.

![Widgets](/images/posts/04_ios_widget_example.png)

<div align="center">
<blockquote class="twitter-tweet"><p lang="en" dir="ltr">The fact that your apps can now take up to 12 icons spaces with a widget make it the most important iOS feature since forever. It&#39;ll give insane engagement and usage to apps that provide valuable widgets. This is the biggest feature in iOS history for apps developers IMO.</p>&mdash; Thomas Ricouard (@Dimillian) <a href="https://twitter.com/Dimillian/status/1275776420686553089?ref_src=twsrc%5Etfw">June 24, 2020</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>
</div>

## Things to consider

* Widgets are powered by SwiftUI, building one with UIKit is out of equation;
* They are extensions of your application, building a standalone widget is not possible;
* You can't interact with a widget other than tapping on it to launch your application - provide read-only information.

## What we will build

We'll assume my banking app has an open API which provides my list of accounts - in banking terms, products - in the following format. Where `type` is an enumeration of values `"main"` and `"secondary"`.

```json
    [{
      "balance": "4675",
      "IBAN": "NL27AAAA0352734310",
      "currency": "EUR",
      "id": "8ab78900f99013a",
      "name": "Main Account"
      "type": "main"
    }]

```

Considering we can only have one main account, our widget will show the following details of this account:
```
	Name
	CurrencySign Balance

	When it was last updated
	SF Symbol (why not?! 🙂)
```

### Our banking app

As a widget can't exist standalone, I've previously built a SwiftUI application to list my accounts - not just the main one. It doesn't do much other than allowing me to show an accounts upon tapping on it and hidding it by tapping again.

![Accounts app](/images/posts/04_ios_widget_app.gif)

### Setup

We now start by creating a new target of Widget Extension.

![Widget Target](/images/posts/04_ios_widget_new_target.png)

During this process, I've named my Widget `CurrentAccount` and disabled `Include Configuration Intent` - as I don't want users to configure/edit the widget but read information from the main account only.

This process will create my target and add the following folder for my project

![Widget Folder](/images/posts/04_ios_widget_folder.png)

### Now comes the fun part

In `CurrentAccount.swift` we already have everything we need, a `TimelineProvider`, a `TimelineEntry`, a `PlaceholderView`, a `EntryView` and our `Widget`.

Let's first go through every component in order to understand what they do and why we need them.

#### TimelineEntry

```swift
struct SimpleEntry: TimelineEntry {
  public let date: Date
}
```

A widget receives information in time. This information is passed via a timeline entry, which by default needs a date where WidgetKit will render the widget.

Our Widget needs more than that. We'll create another type of entry which contains our main account (product)

```swift
struct AccountEntry: TimelineEntry {
  public let date: Date
  public let product: Product
}
```

Where `Product` is a simple struct shared from our main target

```swift
enum ProductType {
  case main, secondary
}

struct Product: Identifiable {
  let id: String
  let name: String
  let balance: Double
  let iban: String
  let type: ProductType
}
```

#### TimelineProvider

Now that we have an entry, we need a way to provide this to the widget through time. The timeline provider is responsible for this.

```swift
struct Provider: TimelineProvider {
  public typealias Entry = SimpleEntry

  public func snapshot(with context: Context, completion: @escaping (SimpleEntry) -> ()) {
    /* implementation */
  }

  public func timeline(with context: Context, completion: @escaping (Timeline<Entry>) -> ()) {
    /* implementation */
  }
}
```

Here you notice it contains two methods, `snapshot` and `timeline`. The first is used to configure/render the widget in transient circumstances, think of it as a demonstration with mock data. We'll therefore implement it like so:

```swift
public func snapshot(with context: Context, completion: @escaping (AccountEntry) -> ()) {
  let previewProduct = Product(
    id: "123a67cf",
    title: "Main Account",
    amount: 280.25,
    iban: "NL27AAA0726252510"
  )
  let entry = AccountEntry(date: Date(), product: previewProduct)
  completion(entry)
}
```

`timeline` is the method used for the real implementation. It will return on completion an array of your entries to be rendered at a given time. WidgetKit will be able to request multiple timelines, so it's not necessary to return multiple values in this array if your content is dynamically fetched.

```swift
public func timeline(with context: Context, completion: @escaping (Timeline<Entry>) -> ()) {
  var entries: [AccountEntry] = []

  /* ... */

  let timeline = Timeline(entries: entries, policy: .atEnd)
  completion(timeline)
}
```

One important thing in the code above is the `policy` expected when initializing a `Timeline`. From the documentation we see: `The policy that determines the earliest date and time WidgetKit requests a new timeline from a timeline provider`. It's a reload policy which can be:

* `atEnd` - Specifies that WidgetKit requests a new timeline after the last date in a timeline passes
* `never` - Specifies that WidgetKit shuold never request a new timeline. As an option, the application can still prompt WidgetKit when new timelines are available
* `after(_ date: Date)` - Pre defines a future date (from the last date in a timeline) that WidgetKit shall request a new one.

Nowing this and since we retrieve a list of accounts and its data for the given moment, our timeline will only return one entry. We'll make use of policy `after(_ date: Date)` to specify how many minutes after rendering we shall request a new timeline, therefore making a new request to fetch our data.

We'll use the current date for the current entry and a future date - 10 minutes apart - where the next timeline will be requested

```swift
let currentDate = Date()
let futureDate = Calendar.current.date(byAdding: .minute, value: 10, to: currentDate)!
```

We'll have to fetch new data in order to create our entry, since a `Product` is required. Having our entry, we can create a timeline and complete our implementation

```swift
viewModel.fetchProducts() { products in
  guard let mainProduct = products.first(where: { $0.type == .main }) else { return }

  let entry = AccountEntry(date: currentDate, product: mainProduct)
  let timeline = Timeline(entries: [entry], policy: .after(refreshDate))
  completion(timeline)
}
```

Our entire `timeline` method looks like:

```swift
public func timeline(with context: Context, completion: @escaping (Timeline<Entry>) -> ()) {
  let currentDate = Date()
  let futureDate = Calendar.current.date(byAdding: .minute, value: 10, to: currentDate)!

  viewModel.fetchProducts() { products in
    guard let mainProduct = products.first(where: { $0.type == .main }) else { return }

    let entry = AccountEntry(date: currentDate, product: mainProduct)
    let timeline = Timeline(entries: [entry], policy: .after(refreshDate))
    completion(timeline)
  }
}
```

Our widget will receive a new timeline - containing 1 entry - every 10 minutes.

#### Widget

The widget is very straight forward. We'll have to configure minor details, such as display name and description.

```swift
@main
struct MainAccount: Widget {
  private let kind: String = "MainAccount"

  public var body: some WidgetConfiguration {
    StaticConfiguration(kind: kind, provider: Provider(), placeholder: PlaceholderView()) { entry in
      MainAccountEntryView(entry: entry)
    }
    .configurationDisplayName("Main Account")
    .description("See information about your main account.")
  }
}
```

Note that this Widget uses a `StaticConfiguration`, as we don't want to allow users to configure/edit it. For a configurable widget, there is `IntentConfiguration`.

#### Views

As you might have seen, we have a `PlaceholderView()` and a `MainAccountEntryView`, that this widget uses. Both are pure SwiftUI views already provided during setup.

```swift
struct PlaceholderView : View {
  var body: some View {
    Text("Placeholder View")
  }
}

struct MainAccountEntryView : View {
  var entry: Provider.Entry

  var body: some View {
    Text(entry.date, style: .time)
  }
}
```

`PlaceholderView` - if the name isn't clear enough - is used when there is no timeline/entry to render. You can use it to display a generic message.

`MainAccountEntryView` is used to render your entries, here you want to display the information needed. It contains an `entry` property which, based on our Provider, contain a `Product`. We could therefore use product's information. Example:

```swift
struct MainAccountEntryView : View {
  var entry: Provider.Entry

  var body: some View {
    VStack(alignment: .leading, spacing: 2) {
      Text(entry.product.name)
      Text(entry.product.iban)
      Text(entry.product.formattedAmount)
    })
  }
}
```

#### Previews

As for any other SwiftUI view, we can also preview them in Xcode Canvas. We create a `PreviewProvider` to show multiple previews with a `Group`, one for our placeholder and another for our entry view.

```swift
struct Previews: PreviewProvider {
  static var previews: some View {
    Group {
      MainAccountEntryView(entry: previewEntry)
            .previewContext(WidgetPreviewContext(family: .systemSmall))
            
      PlaceholderView()
            .previewContext(WidgetPreviewContext(family: .systemSmall))
    }
  }
}
```

You can potentially preview multiple `MainAccountEntryView` using different preview contexts, for instance, for the different widget sizes, `.systemSmall`, `.systemMedium` and `.systemLarge`.

![Widget Preview](/images/posts/04_ios_widget_preview.png)

#### Entry View

You have the freedom - and restrictions 😅 - of SwiftUI to build your placeholder and entry views. Therefore, the focus here isn't to go in depth there, but this is what I've used for the entry view:

```swift
ZStack(alignment: .topLeading) {
  RoundedRectangle(cornerRadius: 0, style: .continuous)
    .fill(Color("widgetBackgroundColor"))

  HStack(alignment: .center, spacing: 8) {
    RoundedRectangle(cornerRadius: 3, style: .continuous)
      .fill(LinearGradient(
          gradient: .init(colors: [.blue, .green]),
          startPoint: .init(x: 0, y: 0),
          endPoint: .init(x: 1, y: 1)
      ))
      .frame(width: 6, height: 120)
      .padding(.leading, -6)

    VStack(alignment: .leading, spacing: 2) {
      Text(entry.product.name)
        .bold()
        .font(Font.system(size: 14))
      Text("\(entry.product.formattedAmount)")
        .bold()
        .lineLimit(1)
        .font(Font.system(size: 22))
      Spacer()
      Text("Last checked at")
        .font(.footnote)
      Text(entry.date, style: .time)
        .font(.footnote)
      Spacer()
      Image(systemName: "creditcard.fill")
        .padding(.bottom, 4)
    }
  }
  .padding(.all, 16)
  .foregroundColor(Color("widgetBackgroundColor"))
}
```

The color sets `widgetBackgroundColor` and `widgetBackgroundColor` have been added to the Assets and support dark mode 🌙

#### Supporting Different Sizes

Our widget is now ready in all 3 available sizes. What if you want to change that?
It's fairly easy with `supportedFamilies`. Let's add that to our widget:


```swift
@main
struct MainAccount: Widget {
  private let kind: String = "MainAccount"

  public var body: some WidgetConfiguration {
    StaticConfiguration(kind: kind, provider: Provider(), placeholder: PlaceholderView()) { entry in
      MainAccountEntryView(entry: entry)
    }
    .configurationDisplayName("Main Account")
    .description("See information about your main account.")
    .supportedFamilies([.systemSmall, .systemMedium])
  }
}
```


#### Our widget in action

![Widget Preview](/images/posts/04_ios_widget_demo.gif)

## Conclusion

Widgets in the home screen is a powerful feature of iOS 14. We've seen how easy it is to create one and improve the user experience of our users. An addition to a static widget is to make it configurable - say, if I'd allow the users to select which account they want to see instead of always the main account, this is possible with the power of intents and an `IntentConfiguration`, which I'll try to cover later.

![widget](/images/posts/04_ios_widget.png)

