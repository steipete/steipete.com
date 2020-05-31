---
layout: post
title:  "InterposeKit - Elegant swizzling in Swift"
date:   2020-05-31 12:00:00 +0200
tags: iOS development
image: /assets/img/2020/interposekit/interposekit-code.png
---

I built a thing! [InterposeKit](https://github.com/steipete/InterposeKit) is a modern library to swizzle elegantly in Swift. It is fully written in Swift 5.2+ and works on `@objc dynamic` Swift functions or Objective-C instance methods.

Since everything's [much better declared on the project website](http://interposekit.com/), I'm just writing some random thoughts that didn't the readme on building this.

## GitHub Actions

Wow! I didn't had the time to play with this before, and damn this is 💖 good. GitHub Action is fast, easy to set up, reliable and superbly well integrated, down to automatic badges for CI state.

There are a few annoyances, like now being able to run Docker containers on macOS (this isn't technical, that's just [a money thing](https://github.community/t/why-is-docker-not-installed-on-macos/17017/2)). The default Jazzy setup to generate documentation runs via Docker, so I had to [go through some hoops](https://github.com/steipete/InterposeKit/blob/85f6c2dcc465811048cac0b31c4edc8bb71d4268/Sources/InterposeKit/InterposeKit.swift#L305-L319) to make my project compile on Linux.

## Swift and Type Aliases

It's extremely unfortunate that the `@convention()` modifier can't be used on existing type aliases - this would have made Interpose way more convenient. I'm honestly tempted to write a proposal to get this into Swift. Both because this would be cool, and because I'd be really interested in the learning experience.

{% twitter https://twitter.com/steipete/status/1266799174563041282 %}

## Swift Package Manager

I finally watched [WWDC2019:410 Creating Swift Packages](https://developer.apple.com/videos/play/wwdc2019/410/) and [WWDC2019:408 Adopting Swift Packages in Xcode](https://developer.apple.com/videos/play/wwdc2019/408/) (Hi Boris!) and I really like SwiftPM. Yes, there's a still a lot to do, but it's getting [Package Resources (SE-0271)](https://github.com/apple/swift-evolution/blob/master/proposals/0271-package-manager-resources.md) with Xcode 12 and the integration with GitHub is nice. Not being allowed to [delete `DerivedData` anymore](https://www.jessesquires.com/blog/2020/02/24/replacing-cocoapods-with-swiftpm/) will be difficult though.

## Class Load Events

`objc_addLoadImageFunc ` is a [big no-no](https://twitter.com/steipete/status/1266464092082114561?s=21), and probably [shouldn't even exist in the header](https://twitter.com/jckarter/status/1266466247748677632?s=21) at all. There's `_dyld_register_func_for_add_image` however, which is great. This includes a C callback, and while Swift does a really great  job to make it just blend into the language, this callback is not a block and can't capture state. I eventually found out that I can put everything [into a struct as long as it's all static](https://github.com/steipete/InterposeKit/blob/85f6c2dcc465811048cac0b31c4edc8bb71d4268/Sources/InterposeKit/InterposeKit.swift#L259-L293), to not have this pollute my global namespace.

Why would I need this? There was [a particular bug in Mac Catalyst](/posts/mac-catalyst-crash-hunt/) that required such a trick.

## Swift 5.2 callAsFunction

Well well well... here I was [bitchin'](https://twitter.com/steipete/status/1227191768153829376?s=20) about Swift getting useless features, only to be extremely happy about `callAsFunction` a few months later. [It even does generics!](https://github.com/steipete/InterposeKit/blob/85f6c2dcc465811048cac0b31c4edc8bb71d4268/Sources/InterposeKit/InterposeKit.swift#L175-L178)

## imp_implementationWithBlock

`imp_implementationWithBlock` has [no way to undo](https://github.com/steipete/InterposeKit/blob/85f6c2dcc465811048cac0b31c4edc8bb71d4268/Sources/InterposeKit/InterposeKit.swift#L130) or deregister the IMP; so once you submitted a block that captures state, you have a permanent memory leak? Oh well.

# Closing Thoughts

This is my first Swift-specific open source project, apart from the usual gists. Please BE harsh on me, I'd like to learn, I had a lot of fun and built this in a weekend, and it helped me to forget time and space (and current world events) for a bit and just thinker. I'm also sure I got things wrong, so please do tell me what can be made better.