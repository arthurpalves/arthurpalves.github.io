---
layout:     post
title:      Allow User Interactivity with an iOS 14 Widget
date:       2020-06-29
summary:    It isn't enough to just create a widget, sometimes we must make it configurable. Here we'll make sure the user has power to mask sentitive data from his account balance.
tags: 		ios14 swiftlang development swift ios widget
categories: tutorial
featured_image: /images/posts/05_configurable_ios_widget.png
---

Last week I've posted about [Building an iOS 14 Widget to Show Account Balance](https://arthuralves.com/tutorial/2020/06/24/building-an-ios14-widget). Some of the concerns around this feature was regarding security. **How can we make sure the user is in control of either showing or masking the account's balance in their home screen?** We can do this by making an Intent Configurable widget instead of a static one.

<div align="center">
 <img src="/images/posts/05_configurable_ios_widget_demo.gif" width="40%">
</div>

Nothing from last week's post is lost, we'll only have to make some small adaptations. We start with an [Intent Definition](https://developer.apple.com/documentation/sirikit/adding_user_interactivity_with_siri_shortcuts_and_the_shortcuts_app).

## Custom Intents

> "By adding custom intents with parameters to your app, you can offer shortcuts that support conversational interaction in Siri and user customization in the Shortcuts app.""
>
> -Apple

Intent Definitions are used to add user interactivity to Siri shortcuts and the Shortcuts app. Now, intents also power the same interactivity with iOS 14 widgets.

Therefore, using the same project, inside my Widget's target, I'll create a new file of type `.intentdefinition`. I'll call it `MainAccountIntent`.

![Main Account Intent](/images/posts/05_configurable_ios_widget_intent_01.png)

I'll make sure this intent contains a few things:

1. A boolean parameter - I call it `maskSensitiveData`;
2. The category of our widget is `View`;
3. `Intent eligible for widgets` is checked;
4. Default value is set to `true` - Better be cautious;
5. And most important, `User can edit value in Shortcups, widgets and Add to Siri`.

![Main Account Intent Filled](/images/posts/05_configurable_ios_widget_intent_02.png)

Our Intent Definition is ready 👏
Now comes the coding part.

## Adjusting the Widget

We'll make a couple changes to our widget, more specifically changing its configuration from `StaticConfiguration` to `IntentConfiguration`. This will require us to change also the `TimelineProvider` to be of type `IntentTimelineProvider`. Our timeline Entry will also be modified to contain a boolean property that reflects our configuration.

### AccountEntry

Our entry is still responsible for carrying data to our view, therefore it's the right place to also carry the information either the user wants to mask sensitive data or not. We'll simply add a property of type boolean to it.

<script src="https://gist.github.com/arthurpalves/ba378f37a39fe7a1300e4162572b3701.js"></script>

### IntentTimelineProvider

We'll start by changing our `Provider`. Complying to the slight different protocol `IntentTimelineProvider` ensures our timeline has access to the current configuration. In our case, if either the uses would like to see sensitive data or not.
Here is what it looks like from last week:

<script src="https://gist.github.com/arthurpalves/b1c16bcb3a7667ee243a926120390eb0.js"></script>

Let's go ahead and change its protocol from `TimelineProvider` to `IntentTimelineProvider`. You'll notice that `typealias Intent` is now a requirement. It will be an alias to our `MainAccountIntent`.
Another thing you'll notice are the changes in the required methods, both `snapshop()` and `timeline()` gain an additional parameter right at the beginning.

<script src="https://gist.github.com/arthurpalves/04dc79827dd14a92dfc30800e213fd1b.js"></script>

Now we just need to initialize our entries with the boolean property coming from our configuration/intent.
As seen last week, `snapshop()` is purely used for transient states, to mock the view of your widget. For this purpose we don't want to mask data, we can simply pass `false` when initializing the entry here.

The real configuration will be passed in our `timeline()`. Below you can see the addition on line 10, which passes `configuration.maskSensitiveData` as boolean value to our entry or fallback to `true`.

<script src="https://gist.github.com/arthurpalves/53d6ee5477fa9a7a44e37a9706ee5238.js"></script>

### Our beautiful view

All set. Our intent is working, the widget is configurable, our entries are carrying the correct value around. Now we must only update our view.

Here I just pass `maskSensitiveData` from our entry to a method that will replace all digits by `*`.

<script src="https://gist.github.com/arthurpalves/1988ee8f76485efd993c019f29741d9c.js"></script>
