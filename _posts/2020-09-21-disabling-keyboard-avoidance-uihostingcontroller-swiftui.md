---
layout: post
title: "Disabling Keyboard Avoidance in SwiftUI UIHostingController"
date: 2020-09-21 21:00:00 +0200
tags: iOS development
image: /assets/img/2020/uihostingcontroller-keyboard/header.png
description: "UIHostingController has logic to avoid the keyboard, which often is unwanted. We explore a hack to disable this feature."
---

<style type="text/css">
div.post-content > img:first-child { display:none; }
</style>

While SwiftUI is still being [cooked hot](/posts/state-of-swiftui/), it's already really useful and can replace many parts of your app. With `UIHostingController` it can be easily mixed with existing UIKit code. With iOS 14, the SwiftUI-team at Apple added keyboard avoidance logic to the hosting controller, which can result in pretty ugly scrolling behavior.

## When Keyboard Avoidance Is Unwanted

When they keyboard is visible and `UIHostingController` doesn't own the full screen, views try to move away from the keyboard.[^2] This has been [quite a frustrating bug for many](https://developer.apple.com/forums/thread/658432), and is especially bad if you [embed `UIHostingController` as table](https://noahgilmore.com/blog/swiftui-self-sizing-cells/)- or collection view cells[^4].

{% twitter https://twitter.com/thesamcoe/status/1306350596715282434?s=20 %}

While it seems that there are some [weird workarounds if you use iOS 14.2](https://twitter.com/zntfdr/status/1306913858263552001?s=21), this seems unreliable, and folks still need a solution for iOS 14.0.

## Fixing Strategies

While I've not been directly affected by this issue, I've been curious and tried to fix this a while back already. My first attempt was adding `.ignoresSafeArea(.keyboard)` to the SwiftUI view, however this doesn't seem to change anything.

Looks like we need the heavy weapons! My usual strategy is to [inspect the view controller's methods](https://twitter.com/steipete/status/1306153060700426240?s=21) and look for something to poke at. This however became much harder with Swift. With a few tricks we can still inspect `UIHostingController` via the Objective-C runtime, however we only see methods that are subclassed from `UIViewController` or exposed with `@objc`. 

There's really nothing interesting in there that we could poke at. A few days later I've been reading [Samuel Défago's brilliant blog post how he wrapped `UICollectionView` for Swift](https://defagos.github.io/swiftui_collection_intro/). In  Part 3 he presents a fix to an issue with `safeAreaInsets` in `UIHostingController`, by [modifying the view class](https://defagos.github.io/swiftui_collection_part3/). This motivated me to take a closer look at the view - maybe Apple was hiding the keyboard avoidance logic there?

```
po SwiftUI._UIHostingView<KeyboardSwiftUIBug.ContentView>.self.perform("_shortMethodDescription")

▿ Optional<Unmanaged<AnyObject>>
  ▿ some : Unmanaged<AnyObject>
    - _value : <_TtGC7SwiftUI14_UIHostingViewV18KeyboardSwiftUIBug11ContentView_: 0x7fff87103a28>:
in _TtGC7SwiftUI14_UIHostingViewV18KeyboardSwiftUIBug11ContentView_:
	Properties:
		@property (nonatomic, readonly) struct UIEdgeInsets safeAreaInsets;
		@property (nonatomic, retain) UIColor* backgroundColor;
		@property (nonatomic, readonly) unsigned long _axesForDerivingIntrinsicContentSizeFromLayoutSize;
		@property (nonatomic, readonly) BOOL _layoutHeightDependsOnWidth;
	Instance Methods:
		- (void) dealloc; (0x7fff566e9320)
		- (id) initWithCoder:(id)arg1; (0x7fff566e91e0)
		- (id) backgroundColor; (0x7fff566eaab0)
		- (void) setBackgroundColor:(id)arg1; (0x7fff566eab30)
		- (id) initWithFrame:(struct CGRect)arg1; (0x7fff566ec710)
		- (void) layoutSubviews; (0x7fff566ea040)
		- (void) traitCollectionDidChange:(id)arg1; (0x7fff566ea630)
		- (struct CGSize) sizeThatFits:(struct CGSize)arg1; (0x7fff566eadb0)
		- (struct UIEdgeInsets) safeAreaInsets; (0x7fff566ea790)
		- (void) didUpdateFocusInContext:(id)arg1 withAnimationCoordinator:(id)arg2; (0x7fff566ec230)
		- (id) preferredFocusEnvironments; (0x7fff566ec150)
		- (void) safeAreaInsetsDidChange; (0x7fff566ea6c0)
		- (void) didMoveToSuperview; (0x7fff566e9df0)
		- (void) _geometryChanged:(void*)arg1 forAncestor:(id)arg2; (0x7fff566e9ef0)
		- (id) _childFocusRegionsInRect:(struct CGRect)arg1 inCoordinateSpace:(id)arg2; (0x7fff566ec080)
		- (struct ?) _baselineOffsetsAtSize:(struct CGSize)arg1; (0x7fff566eacd0)
		- (unsigned long) _axesForDerivingIntrinsicContentSizeFromLayoutSize; (0x7fff566eabd0)
		- (void) contentSizeCategoryDidChange; (0x7fff566ea5c0)
		- (void) keyboardWillShowWithNotification:(id)arg1; (0x7fff566ec4f0)
		- (void) keyboardWillHideWithNotification:(id)arg1; (0x7fff566ec5a0)
		- (void) externalEnvironmentDidChange; (0x7fff566edd40)
		- (id) makeViewDebugData; (0x7fff566ec5f0)
		- (void) _geometryChanges:(id)arg1 forAncestor:(id)arg2; (0x7fff566e9e20)
(UIView ...)
```

This looks interesting[^3]. There's quite a few Swift methods that have been exposed to the ObjC runtime. `keyboardWillShowWithNotification:` and `keyboardWillHideWithNotification:` look exactly like candidates to tweak. We're lucky here that the SwiftUI engineers didn't use the block-based NSNotification-API[^1] but used the target/selector approach - which does need `@objc` annotations to work.

## Subclassing at Runtime

We want to replace the implementation of `keyboardWillShowWithNotification:` with an empty one. The classic solution here would be swizzling, however that would modify *all* instances of `UIHostingController`, and we don't know if the view class isn't used somewhere else. This might work but seems risky.

A better strategy is to modify only instances we control. We can do that via dynamic subclassing. It's my favorite way to modify behavior on a per-object basis, in fact I wrote a whole Swift library called [InterposeKit](https://interposekit.com/) to make this easy:

```swift
import SwiftUI
import InterposeKit

extension UIHostingController {
convenience public init(rootView: Content, ignoresKeyboard: Bool) {
    self.init(rootView: rootView)

    if ignoreKeyboard {
        _ = try? self.view.hook(NSSelectorFromString("keyboardWillShowWithNotification:")) { (
            store: TypedHook<@convention(c) (AnyObject, Selector, AnyObject) -> Void,
                             @convention(block) (AnyObject, AnyObject) -> Void>) in { _, _ in }
        }
    }
}
}
```

Dynamic subclassing isn't very tricky, but the challenge is to write it in a way where it fails gracefully if the private API we modify here is changed. InterposeKit adds a lot of error handling, next to a convenient API so you make fewer mistakes and have a more stable app. It will throw an error if the selection no longer exists or has a different type than the one you expect.

We can achieve something similar using built-in methods:

```swift
extension UIHostingController {
convenience public init(rootView: Content, ignoresKeyboard: Bool) {
    self.init(rootView: rootView)

    if ignoresKeyboard {
        guard let viewClass = object_getClass(view) else { return }

        let viewSubclassName = String(cString: class_getName(viewClass)).appending("_IgnoresKeyboard")
        if let viewSubclass = NSClassFromString(viewSubclassName) {
            object_setClass(view, viewSubclass)
        }
        else {
            guard let viewClassNameUtf8 = (viewSubclassName as NSString).utf8String else { return }
            guard let viewSubclass = objc_allocateClassPair(viewClass, viewClassNameUtf8, 0) else { return }

            if let method = class_getInstanceMethod(viewClass, NSSelectorFromString("keyboardWillShowWithNotification:")) {
                let keyboardWillShow: @convention(block) (AnyObject, AnyObject) -> Void = { _, _ in }
                class_addMethod(viewSubclass, NSSelectorFromString("keyboardWillShowWithNotification:"),
                                imp_implementationWithBlock(keyboardWillShow), method_getTypeEncoding(method))
            }
            objc_registerClassPair(viewSubclass)
            object_setClass(view, viewSubclass)
        }
    }
}
}
```

See [my gist](https://gist.github.com/steipete/da72299613dcc91e8d729e48b4bb582c#file-uihostingcontroller-keyboard-swift) for a version that also removes the `safeAreaInsets`. Who would have thought that runtime trickery is still useful in SwiftUI times?

If this is useful to you, ping [@steipete on Twitter](https://twitter.com/steipete)!

[^1]: The block-based notification API nowadays is inconvenient as it doesn't automatically deregister observers - using the target/action one is simpler, as these observers automatically deregister since iOS 9.

[^2]: If you wanna try this for yourself, [I've prepared an example here](https://twitter.com/steipete/status/1306925835010609152?s=21).

[^3]: The method `makeViewDebugData` also looks pretty interesting...

[^4]: Apple Folks: FB8698723 - Provide API in UIHostingController to disable keyboard avoidance for SwiftUI views.
