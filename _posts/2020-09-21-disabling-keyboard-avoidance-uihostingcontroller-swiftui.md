---
layout: post
title: "Disabling Keyboard Avoidance in SwiftUI UIHostingController"
date: 2020-09-21 19:00:00 +0200
tags: iOS development
image: /assets/img/2020/swift-logging/logd.jpeg
description: "UIHostingController has logic to avoid the keyboard, which often is unwanted. We explore a hack to disable this feature."
---

<style type="text/css">
div.post-content > img:first-child { display:none; }
</style>

The beauty of SwiftUI is that it can be easily mixed with existing UIKit code via `UIHostingController`. With iOS 14, Apple added keyboard avoidance logic to the hosting controller, which often results in pretty ugly bugs. 

## When Keyboard Avoidance Is Unwanted

When they keyboard is visible and `UIHostingController` doesn't own the full screen, views wobble around:

{% twitter https://twitter.com/thesamcoe/status/1306350596715282434?s=20 %}

This has been quite a frustrating bug for many. And while it seems that there are some [weird workarounds if you use iOS 14.2](https://twitter.com/zntfdr/status/1306913858263552001?s=21), this seems unreliable, and folks still need a solution for iOS 14.0.

## Fixing Strategies

While I've not been directly affected by this issue, I've been curious and tried to fix this a while back already. The usual strategy is to inspect the view controller's methods. This however became much harder with Swift.

With a few tricks we can still inspect `UIHostingController` via the Objective-C runtime, however we only see methods that are subclassed from `UIViewController` or exposed with `@objc`. 

{% twitter https://twitter.com/steipete/status/1306153060700426240?s=21 %}

There's really nothing interesting in there that we could poke at. A few days later I've been reading [Samuel Défago's brilliant blog post how he wrapped `UICollectionView` for Swift](https://defagos.github.io/swiftui_collection_intro/). In  Part 3 he presents a fix to an issue with `safeAreaInsets` in `UIHostingController`, by [modifying the view class](https://defagos.github.io/swiftui_collection_part3/). This motivated me to take a closer look at the view - maybe Apple was hiding the keyboard avoidance logic there?

```
po SwiftUI._UIHostingView<KeyboardSwiftUIBug.ContentView>.self.perform("_shortMethodDescription")

▿ Optional<Unmanaged<AnyObject>>
  ▿ some : Unmanaged<AnyObject>
    - _value : <_TtGC7SwiftUI14_UIHostingViewV18KeyboardSwiftUIBug11ContentView_: 0x7fff87103a28>:
in _TtGC7SwiftUI14_UIHostingViewV18KeyboardSwiftUIBug11ContentView_:
	Properties:
		@property (nonatomic) struct CGRect frame;
		@property (nonatomic) struct CGRect bounds;
		@property (nonatomic, readonly) struct UIEdgeInsets safeAreaInsets;
		@property (nonatomic, retain) UIColor* backgroundColor;
		@property (nonatomic, readonly) unsigned long _axesForDerivingIntrinsicContentSizeFromLayoutSize;
		@property (nonatomic, readonly) BOOL _layoutHeightDependsOnWidth;
		@property (nonatomic, copy) NSArray* accessibilityElements;
		@property (nonatomic, readonly) BOOL canBecomeFirstResponder;
		@property (nonatomic, readonly) NSArray* preferredFocusEnvironments;
	Instance Methods:
		- (void) dealloc; (0x7fff566e9320)
		- (BOOL) respondsToSelector:(SEL)arg1; (0x7fff566eb650)
		- (id) forwardingTargetForSelector:(SEL)arg1; (0x7fff566ebe00)
		- (id) initWithCoder:(id)arg1; (0x7fff566e91e0)
		- (void) .cxx_destruct; (0x7fff566e9340)
		- (struct CGRect) bounds; (0x7fff566ea330)
		- (void) setBounds:(struct CGRect)arg1; (0x7fff566ea430)
		- (struct CGRect) frame; (0x7fff566ea2f0)
		- (id) backgroundColor; (0x7fff566eaab0)
		- (void) setBackgroundColor:(id)arg1; (0x7fff566eab30)
		- (void) setFrame:(struct CGRect)arg1; (0x7fff566ea310)
		- (id) initWithFrame:(struct CGRect)arg1; (0x7fff566ec710)
		- (void) layoutSubviews; (0x7fff566ea040)
		- (void) traitCollectionDidChange:(id)arg1; (0x7fff566ea630)
		- (struct CGSize) sizeThatFits:(struct CGSize)arg1; (0x7fff566eadb0)
		- (struct UIEdgeInsets) safeAreaInsets; (0x7fff566ea790)
		- (id) accessibilityElements; (0x7fff566eae10)
		- (void) setAccessibilityElements:(id)arg1; (0x7fff566eb210)
		- (void) didUpdateFocusInContext:(id)arg1 withAnimationCoordinator:(id)arg2; (0x7fff566ec230)
		- (id) preferredFocusEnvironments; (0x7fff566ec150)
		- (void) tintColorDidChange; (0x7fff566edd30)
		- (void) didMoveToWindow; (0x7fff566e9d60)
		- (BOOL) canPerformAction:(SEL)arg1 withSender:(id)arg2; (0x7fff566eb780)
		- (BOOL) canBecomeFirstResponder; (0x7fff566eb550)
		- (void) safeAreaInsetsDidChange; (0x7fff566ea6c0)
		- (void) didMoveToSuperview; (0x7fff566e9df0)
		- (void) _geometryChanged:(void*)arg1 forAncestor:(id)arg2; (0x7fff566e9ef0)
		- (id) _childFocusRegionsInRect:(struct CGRect)arg1 inCoordinateSpace:(id)arg2; (0x7fff566ec080)
		- (struct ?) _baselineOffsetsAtSize:(struct CGSize)arg1; (0x7fff566eacd0)
		- (id) targetForAction:(SEL)arg1 withSender:(id)arg2; (0x7fff566eb9c0)
		- (BOOL) _layoutHeightDependsOnWidth; (0x7fff566eabe0)
		- (unsigned long) _axesForDerivingIntrinsicContentSizeFromLayoutSize; (0x7fff566eabd0)
		- (void) contentSizeCategoryDidChange; (0x7fff566ea5c0)
		- (void) keyboardWillShowWithNotification:(id)arg1; (0x7fff566ec4f0)
		- (void) keyboardWillHideWithNotification:(id)arg1; (0x7fff566ec5a0)
		- (void) externalEnvironmentDidChange; (0x7fff566edd40)
		- (void) accessibilityBooleanDidChange:(id)arg1; (0x7fff566ea6f0)
		- (void) enableAccessibilityNotificationFired:(id)arg1; (0x7fff566ea1e0)
		- (id) makeViewDebugData; (0x7fff566ec5f0)
		- (void) _geometryChanges:(id)arg1 forAncestor:(id)arg2; (0x7fff566e9e20)
(UIView ...)
```

This looks interesting. There's quite a few Swift methods that have been exposed to the ObjC runtime. `keyboardWillShowWithNotification:` and `keyboardWillHideWithNotification:` look exactly like candidates to tweak. We're lucky here that the SwiftUI engineers didn't use the block-based NSNotification-API[^1] but used the target/selector approach - which does need `@objc` annotations to work.

## The Solution is Swizzling

We can simply replace the implementation of `keyboardWillShowWithNotification:` with an empty one, which should stop the keyboard avoidance. We don't wanna modify all  `UIHostingController` instances, only the ones under our control, so we use dynamic subclassing here:

<script src="https://gist.github.com/steipete/da72299613dcc91e8d729e48b4bb582c.js"></script>

We're not even swizzling here - we just create a dynamic subclass at runtime with the new, empty implementation. Who would have thought that runtime trickery is still useful in SwiftUI times? 

Apple Folks: FB8698723 - Provide API in UIHostingController to disable keyboard avoidance for SwiftUI views.

[^1]: The block-based notification API nowadays is inconvenient as it doesn't automatically deregister observers - using the target/action one is simpler, as these observers automatically deregister since iOS 9.

