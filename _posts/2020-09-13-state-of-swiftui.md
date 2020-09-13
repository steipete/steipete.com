---
layout: post
title: "The State of SwiftUI"
date: 2020-09-13 11:00:00 +0200
tags: iOS development SwiftUI
image: /assets/img/2020/fruta-swiftui/fruta-crash.png
description: "Let's look at SwiftUI in iOS 14 and macOS Big Sur via evaluating Apple's Fruta Sample app."
---

<style type="text/css">
div.post-content > img:first-child { display:none; }
</style>

Apple released SwiftUI last year, and it's been a really exciting and wild ride. With iOS 14, a lot of the rough edges have been fixed — is SwiftUI finally ready for production?

## Fruta Sample App

Let's look at [Apple's Fruta example](https://developer.apple.com/documentation/app_clips/fruta_building_a_feature-rich_app_with_swiftui), a cross-platform feature-rich app that's completely built in SwiftUI. It's great that Apple finally releases a more complex application for this year's cycle. I took a look when Big Sur beta 1 came out, and it's been pretty rough:

{% twitter https://twitter.com/steipete/status/1277623561604214784?s=21 %}

Since then there's been many betas and we're nearing the end of the cycle, with the GM expected in October. It's time to look at it again. And indeed the SwiftUI team did a great job at fixing the various issues - toolbar works pretty reliable, the sidebar no longer jumps out, multiple windows works, however [views still sometimes are misaligned](https://twitter.com/steipete/status/1305054121523916806?s=21) and it's still fairly easy to make it crash on both [macOS (FB8682269)](https://twitter.com/steipete/status/1305051342596177921?s=21) and [iOS 14b8 (FB8682290)](https://twitter.com/steipete/status/1305052083989684224?s=21).

## SwiftUI AttributeGraph Crashes

Most SwiftUI crashers are either a diffing issue in the AttributeGraph, or a bug with one of the bindings to the platform controls (AppKit or UIKit). Whenever you see `AG::Graph` in the stack trace that's Swift's AttributeGraph, which takes over representing the view hierarchy and diffing. Crashes there usually are in this form:

```
 Fruta[3607:1466511] [error] precondition failure: invalid size for indirect attribute: 25 vs 24
```

Googling for this error reveals that there's [a](https://github.com/fermoya/SwiftUIPager/issues/60) [lot](https://developer.apple.com/forums/thread/129171) [of](https://stackoverflow.com/questions/58304009/how-to-debug-precondition-failure-in-xcode) [similar](https://www.reddit.com/r/SwiftUI/comments/fosrbf/precondition_failure_invalid_input_index/) [problems](https://twitter.com/steipete/status/1258762457805455361), people sometimes do find workaround via wrapping views into other views or changing the hierarchy. Mostly though you are powerless, this is something Apple needs to fix in their framework. Since SwiftUI ships as part of the OS, end user need to update their devices to get these fixes.

## Platform Binding Crashes

Removing a favorited item while it is selected crashes in the AppKit binding that syncs the SwiftUI state with `NSTableView`:

<blockquote class="twitter-tweet" data-conversation="none" data-dnt="true"><p lang="en" dir="ltr">Removing a favorite crashes the app immediately. <a href="https://t.co/i5wiMJr4XX">pic.twitter.com/i5wiMJr4XX</a></p>&mdash; Peter Steinberger (@steipete) <a href="https://twitter.com/steipete/status/1305075451711369216?ref_src=twsrc%5Etfw">September 13, 2020</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

```
2020-09-13 10:31:25.483965+0200 Fruta[79371:2051792] [General] Row 0 out of row range [0--1] for rowViewAtRow:createIfNeeded:
2020-09-13 10:31:25.498144+0200 Fruta[79371:2051792] [General] (
	0   CoreFoundation                      0x00007fff20cdb0df __exceptionPreprocess + 242
	1   libobjc.A.dylib                     0x00007fff20b3d469 objc_exception_throw + 48
	2   AppKit                              0x00007fff237a7905 -[NSTableRowData rowViewAtRow:createIfNeeded:] + 675
	3   AppKit                              0x00007fff23813008 -[NSTableView viewAtColumn:row:makeIfNecessary:] + 29
	4   SwiftUI                             0x00007fff49565fc7 $s7SwiftUI19ListCoreCoordinatorC17selectionBehavior5atRow2inAA012PlatformItemC0V0L0V09SelectionG0VSgSi_So11NSTableViewCtF + 39
	5   SwiftUI                             0x00007fff49566b36 $s7SwiftUI19ListCoreCoordinatorC18selectionDidChange2inySo11NSTableViewC_tF + 2662
	6   SwiftUI                             0x00007fff49187850 $s7SwiftUI26NSTableViewListCoordinatorC05tableD19SelectionIsChangingyy10Foundation12NotificationVFTm + 112
	7   SwiftUI                             0x00007fff49187912 $s7SwiftUI26NSTableViewListCoordinatorC05tableD19SelectionIsChangingyy10Foundation12NotificationVFToTm + 114
	8   CoreFoundation                      0x00007fff20c56a6c __CFNOTIFICATIONCENTER_IS_CALLING_OUT_TO_AN_OBSERVER__ + 12
	9   CoreFoundation                      0x00007fff20cf23bb ___CFXRegistrationPost_block_invoke + 49
	10  CoreFoundation                      0x00007fff20cf232f _CFXRegistrationPost + 454
	11  CoreFoundation                      0x00007fff20c275ce _CFXNotificationPost + 723
	12  Foundation                          0x00007fff218ba5e2 -[NSNotificationCenter postNotificationName:object:userInfo:] + 59
	13  AppKit                              0x00007fff237ae588 -[NSTableView _sendSelectionChangedNotificationForRows:columns:] + 219
	14  AppKit                              0x00007fff2373b739 -[NSTableRowData _updateVisibleViewsBasedOnUpdateItems] + 4503
	15  AppKit                              0x00007fff2373a453 -[NSTableRowData _updateVisibleViewsBasedOnUpdateItemsAnimated] + 224
	16  AppKit                              0x00007fff2371a2dc -[NSTableRowData _doWorkAfterEndUpdates] + 95
	17  AppKit                              0x00007fff2371a19d -[NSTableView _endUpdateWithTile:] + 119
	18  SwiftUI                             0x00007fff49186759 $s7SwiftUI26NSTableViewListCoordinatorC011updateTableD0_4from2toySo0cD0C_xxtF + 1145
	19  SwiftUI                             0x00007fff4956c010 $s7SwiftUI19ListCoreCoordinatorC29updateTableViewAndVisibleRows_4from2toySo07NSTableH0C_xxtFyyXEfU_ + 304
	20  SwiftUI                             0x00007fff4956ccf5 $s7SwiftUI19ListCoreCoordinatorC24withSelectionUpdateGuardyyyyXEF + 53
	21  SwiftUI                             0x00007fff4956b2ff $s7SwiftUI19ListCoreCoordinatorC29updateTableViewAndVisibleRows_4from2toySo07NSTableH0C_xxtF + 79
```

There's likely more bugs waiting to be discovered, but I only spent a few hours with Fruta and writing up this article.

## Performance

On my 2,4 GHz 8-Core Intel Core i9 MacBook Pro, changing the selection takes over a second to update the main view. This feels sluggish and it significantly longer than even most websites - that load data via the network - need. Fruta has everything local. What's so slow here? Let's look at Instruments!

![](/assets/img/2020/fruta-swiftui/instruments.png)

* Of the 10 seconds captured, 30% are used for Swift and ObjC's various retain/release and malloc's.
* `NSAttributedString` shows up often in stack traces, giving us a hint that text layout seems especially expensive.
* The AttributeGraph SwiftUI layout engine seems to create a lot of throwaway objects. These might mostly be Swift structs, however they still are expensive.
* JPG decoding happens on the main thread, however is only responsible of less than 1% of the time spent here.
* When checking "Hide System Libraries" there's basically no work done in Fruta's business logic. 
* Sorting for "Top Functions" we see that AppKit's auto layout logic is taking up a lot of time, combined with SwiftUI's graph.
* There seems to be a lot of unnecessary invalidation. For example, the `AppKitToolbarCoordinator` adds a toolbar item, which triggers `NSHostingView.preferencesDidChange()` causing everything to re-layout once again, even though the toolbar size doesn't change.

The good news is, there seem to be a lot of potential future optimizations possible to make this fast. Alternative, there's always the possibility to [drop out of SwiftUI for performance critical parts](https://twitter.com/noahsark769/status/1304938866999046144?s=21).

This is not unique to Fruta, I've been taking a look at [@Dimillian's](https://twitter.com/Dimillian) RedditOS app which is built with SwiftUI on macOS. He [stopped development](https://twitter.com/Dimillian/status/1301802048824979456) because it's so slow that it's not shippable. I did some debugging with an earlier version of Big Sur where the app still somewhat worked:

{% twitter https://twitter.com/steipete/status/1282655123244752897?s=20 %}

The general pattern here seems to be AppKit. The interaction between SwiftUI views and AppKit views [seems to](https://twitter.com/fcbunn/status/1259078251340800000) be [poor](https://twitter.com/stuartcarnie/status/1301895206875181056). It's important to understand that SwiftUI in itself is fast - for many use cases it's faster than even using `CALayer`, [as 
@cocoawithlove did prove](https://twitter.com/cocoawithlove/status/1143859576661393408) - and the UIKit port is by far faster and better than the AppKit port.

## Conclusion

If your target platform is iOS 14, you're now good to go with hobby projects. I personally wouldn't yet use it for production apps, although the crash rate likely is manageable and Apple is actively improving things with every release. Remember that SwiftUI ships with the OS, not with your app, so any bug fixes will only help if your users update the OS.

Other ports are not so great. AppKit seems particularly trouble, but I also heard of big issues with tvOS. If you need to deploy your app to the Mac, use Catalyst, which is a much more stable binding and feels really good with Big Sur's native mode where content is no longer scaled.

If you are curious about SwiftUI, please don't let this stop your enthusiasm. It's clearly the future at Apple and all these issues will be resolved within a few years.