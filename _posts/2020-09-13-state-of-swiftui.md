---
layout: post
title: "The State of SwiftUI"
date: 2020-08-24 19:00:00 +0200
tags: iOS development
image: /assets/img/2020/swift-logging/logd.jpeg
description: "Let's look at SwiftUI via evaluating Apple's Fruta app."
---

<style type="text/css">
div.post-content > img:first-child { display:none; }
</style>

Apple released SwiftUI last year, and it's been a really exciting and wild ride. With iOS 14, a lot of the rough edges have been fixed — is SwiftUI getting ready for production?

## Fruta

Let's look at Apple's Fruta example, a cross-platform feature-rich app that's completely built in SwiftUI. It's great that Apple finally releases a more complex application for this year's cycle. I took a look when Big Sur beta 1 came out, and it's been pretty rough:

{% twitter https://twitter.com/steipete/status/1277623561604214784?s=21 %}

Since then there's been many betas and we're nearing the end of the cycle, with the GM expected in October. It's time to look at it again. And indeed the SwiftUI team did a great job at fixing the various issues - toolbar works pretty reliable, the sidebar no longer jumps out, multiple windows works, however it's still fairly easy to make it crash:

{% twitter https://twitter.com/steipete/status/1305051342596177921?s=21 %}

### SwiftUI AttributeGraph Errors

Whenever you see `AG::Graph` in the stack trace that's Swift's AttributeGraph, which takes over representing the view hierarchy and diffing. Bugs there usually are in this form:

```
 Fruta[3607:1466511] [error] precondition failure: invalid size for indirect attribute: 25 vs 24
```

Googling for this error reveals that there's [a](https://github.com/fermoya/SwiftUIPager/issues/60) [lot](https://developer.apple.com/forums/thread/129171) [of](https://stackoverflow.com/questions/58304009/how-to-debug-precondition-failure-in-xcode) [similar](https://www.reddit.com/r/SwiftUI/comments/fosrbf/precondition_failure_invalid_input_index/) [problems](https://twitter.com/steipete/status/1258762457805455361), people sometimes do find workaround via wrapping views into other views or changing the hierarchy. Mostly though you are powerless, this is something Apple needs to fix in their framework.

## Other Crashes

Removing a Favorited item while it is selected crashes:

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

## Performance

## Distribution

Which brings us to the next problem: distribution. SwiftUI ships with the OS, so any bug fixes require an OS update. Using SwiftUI before iOS 13.4 is extremely rough; if you can drop iOS 13 completely and build for iOS 14 exclusively.