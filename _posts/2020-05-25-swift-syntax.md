---
layout:     post
title:      Building a File Parser with an Abstract Syntax Tree
date:       2020-05-25
summary:    Recently, I came across SwiftSyntax and used it for CoherentSwift. It allows for Swift tools to parse, inspect, generate, and transform Swift source code.
tags: 		swiftlang development swift ios macos cohesion swiftsyntax
categories: tooling
featured_image: /images/posts/03_swift_syntax.png
---

I was familiar with the [Swift Compiler Architecture](https://swift.org/swift-compiler/#compiler-architecture), but only recently I came across [SwiftSyntax](https://github.com/apple/swift-syntax), which - as from it's own description - is:

> "[...] a set of Swift bindings for the [libSyntax](https://github.com/apple/swift/tree/master/lib/Syntax) library. It allows for Swift tools to parse, inspect, generate, and transform Swift source code."

Well, seems exactly what I needed for [CoherentSwift](https://github.com/arthurpalves/coherent-swift) 🎉 

It is used by the Swift Compiler in it's very first task and is responsible for generating an **Abstract Syntax Tree (AST)**, which will be taken down the further steps. I suggest you get familiar to how this process works reading a little about the [compiler architecture](https://swift.org/swift-compiler/#compiler-architecture).

Among the tools that already use SwiftSyntax we find code formatters such as [SwiftRewriter](https://github.com/inamiy/SwiftRewriter), [swift-format](https://github.com/apple/swift-format) and others like [Piranha](https://github.com/uber/piranha) - Uber's own tool to refactor code related to stale flags. One thing they have in common, these tools modify/edit your code.

To help us understand how it works and what you can do with SwiftSyntax we have the post from [Mattt](https://twitter.com/mattt) on [NSHipster](https://nshipster.com/swiftsyntax/), which sums up to:

* [Writing Swift Code: The Hard Way](https://nshipster.com/swiftsyntax/#writing-swift-code-the-hard-way)
* [Rewriting Swift Code](https://nshipster.com/swiftsyntax/#rewriting-swift-code)
* [Highlighting Swift Code](https://nshipster.com/swiftsyntax/#highlighting-swift-code)

I really suggest you take a look at it.

## SyntaxRewriter

As already seen above, most of the tools using SwiftSyntax modify code, which in my view turns out to be the biggest strenght of the lib, due to the fact that these care about snippets of code, one at a time, they go incrementally through pieces of `TokenSyntax` and act differently depending on their `tokenKind`. This is done through `SyntaxRewriter`'s `visit(_:)` methods.

This couldn't be more straight forward! Let's see...
1. Visit a `TokenSyntax`;
2. Given it's kind, do something with it;
3. Return the modified token.

## SyntaxFactory 🏭

This is a set of high level APIs for creating code. Anything, literally anything. It isn't necessarily convenient as you'd have to create syntax one by one. Given the need to create a static property `static let shared = Something()`, we'd make at least 5 calls to the APIs:
1. `makeStaticKeyword`;
2. `makeLetKeyword`;
3. `makeIdentifier`;
4. `makeEqualToken`;
5. `ObjectCreationExpression`

It's at best exhausting to let code write code.

> You can check the available APIs going through all the 5525(**!**) lines of [SyntaxFactory](https://github.com/apple/swift-syntax/blob/master/Sources/SwiftSyntax/gyb_generated/SyntaxFactory.swift)'s code.

## But... Hm... I don't want to edit or write code 🤔

With [CoherentSwift](https://github.com/arthurpalves/coherent-swift) I don't want to create nor edit code, and here SwiftSyntax is extremely helpful - **just not as much as it could be**.

Similar to Rewriter and Factory, we have **SyntaxVisitor**. It allow us to go through `TokenSyntax` (a.k.a walk all nodes of the tree), and by overridding it's `visit(_:)` methods we can parse/analyse given `TokenSyntax`. The visit method's return a `SyntaxVisitorContinueKind`, an enumeration:

<script src="https://gist.github.com/arthurpalves/7c709b162523650c4c8b68195c26b193.js"></script>

See this basically as a `Bool`, for every visit, should it move forward and also pay a visit to it's children or should it stop right now?

### Visits

In AST we have syntaxes for everything, and we have at least double the amount of methods for them, but before going through a few specific examples, let's cover the generic `visit(_:)` method here.

![visit](/images/posts/03_2_generic_visit.png)

This couldn't be more generic, it goes through every node. If you do want to parse them all using something in common, great, and if you want to deal with different `TokenKind` in different ways, a switch case is your friend.

However, we do have `visit(_:)` methods for specific `TokenKind` too.
If we want to parse entire classes:

<script src="https://gist.github.com/arthurpalves/54785868df7d5e50c77939310c56f10e.js"></script>

This give us easy access to the class syntax, with immediate properties such as `identifier`, `attributes`, `modifiers`, `members` and more.

But remember, all this is abstract, they will have more and more abstract syntaxes within them, where at one point we can also find `properties` and `functions`.

### Overriding multiple visits

I find it useful to override multiple `visit(_:)` methods, as I want to process them in complete different ways, i.e.:

<script src="https://gist.github.com/arthurpalves/4f1e3907355631bb034fa3d74070901f.js"></script>

I also want to keep a record of this tree so that I can measure the cohesion of this code, for that I want to know specific things:

1. Which high definitions (Struct, Class) I find in a file
2. Which properties are members of these definitions
3. Which methods are members of these definitions
4. Which properties are members of these methods
5. Which methods are private
6. Which properties are static
7. Which high definition has extensions

This is enough for [CoherentSwift](https://github.com/arthurpalves/coherent-swift) to measure the cohesion for the given code.

> When cohesion is high, it means that the methods and variables of the class are co-dependent and hang together as a logical whole.
> - Clean Code pg. 140 

### Struggles with Parsing

In my recent experience, its extremelly easy to go through the high level tokens within another, but going down the rabbit hole trying to map an entire class at once turned out to be very error prone, for this reason I'm following two approaches at the moment:
- Map immediate members of a definition within the same visit.

	For this purpose, I've created a Factory to give back the expected properties for a given node:
	![class visit](/images/posts/03_3_class_visit.png)

- Expect another visit to a different syntax (i.e.: `FunctionDeclSyntax`)

	Here, the factory also process high level tokens looking for properties and return our very own `CSMethod` after assigning the found `[CSProperty]` to it.
	![method visit](/images/posts/03_4_method_visit.png)

The downside of the last approach is, with all the abstract syntaxes, it is also error prone to climb up the tree as it is climbing down, so I had to keep track of the `currentDefinition` being processed, if any.

## Post Visits

If you've seen enough of `visit(_:)`, how about `visitPost(_:)`?

You read it right, for every *visit& there is a *visitPost*. A post is called after a visit has been paid to the given syntax and all its descendants, it therefore doesn't return any value.
It's very useful for post processing.

<script src="https://gist.github.com/arthurpalves/56fb7427f85260ece995850a4af1d294.js"></script>

## Conclusion

SwiftSyntax is very powerful, not always easy - depending on what you want to achieve, but has certainly increased the accuracy of [CoherentSwift](https://github.com/arthurpalves/coherent-swift)'s measurement, as part of the upcoming **0.5.0 release**.



