---
layout: post
title: "Fixing keyboardShortcut in SwiftUI"
date: 2021-01-31 12:00:00 +0200
tags: iOS SwiftUI development
image: /assets/img/2021/fixing-keyboardshortcut-in-swiftui/header.png
description: "With iOS 14, `keyboardShortcut` was added as a convenient native way to add keyboard shortcuts to SwiftUI. However, if you end up using it, it likely won't work. I've been curious why that is, so let's follow me for a round of SwiftUI debugging! Spoiler: the workaround is at the end of this article."
---

With iOS 14, `keyboardShortcut` was added as a convenient native way to add keyboard shortcuts to SwiftUI. However, if you end up using it, it likely won't work. I've been curious why that is, so let's follow me for a round of SwiftUI debugging! Spoiler: the workaround is at the end of this article.

## Behavior Inventory

Let's first check out how this feature works:

```swift
struct ContentView: View {
    var body: some View {
        Button("Keyboard Enabled Button") {
            print("P pressed")
        }.keyboardShortcut("p", modifiers: [.command])
    }
}
```

Simple enough. However it just didn't work in production. After simplifying my setup and eventually writing my own example project, I've eventually got the realization that my code is correct, and that this just doesn't work. But surely Apple tested this? Let's try a few combinations:

- UIKit app lifecycle, iOS 14: ❌
- UIKit app lifecycle, Catalyst Big Sur: ✅[^1]
- SwiftUI app lifecycle, iOS 14: ✅
- SwiftUI app lifecycle, Catalyst Big Sur: ✅[^1]

[^1]: I've seen issues with keyboard handling in Catalyst, so I recommend testing everything before you rely on this functionality there.

Right. So things work pretty much everywhere, but not in the use case that likely will be most common out there - when mixing SwiftUI and UIKit.

## The Solution

I've been discussing this on Twitter and quickly was [presented with a workaround (thanks Mateus!)](https://twitter.com/1mtsrodrigues/status/1355555597354225665?s=21) that fixes this:

```swift
let host = UIHostingController(rootView: SwiftUIView())

// Workaround for keyboardShortcut not working:
let window = UIWindow()
window.rootViewController = host
window.makeKeyAndVisible()

present(host, animated: true, completion: nil)
```

And sure enough - this indeed does the trick! Now I got very curious. Mateus talks about a `keyboardShortcutBridge` in `UIHostingController` that takes care of keyboard management. Let's look if we can verify that in lldb:

```
Printing description of host:
<_TtGC7SwiftUI19UIHostingControllerV15SwiftUIKeyboard11SwiftUIView_: 0x7fc4f4408360>
(lldb) expression -l objc -O -- [0x7fc4f4408360 _ivarDescription]
<_TtGC7SwiftUI19UIHostingControllerV15SwiftUIKeyboard11SwiftUIView_: 0x7fc4f4408360>:
in _TtGC7SwiftUI19UIHostingControllerV15SwiftUIKeyboard11SwiftUIView_:
	allowedBehaviors (): Value not representable, 
	host (): Value not representable, 
	customTabItem (): Value not representable, 
	toolbarCoordinator (): Value not representable, 
	swiftUIToolbar (): Value not representable, 
	toolbarBridge (): Value not representable, 
	keyboardShortcutBridge (): Value not representable, 
```

Good old `_ivarDescription` is still useful and shows Swift ivars as well, it just can't show the real type, but it's good enough to confirm that there's indeed a `keyboardShortcutBridge`.

## Who sets `keyboardShortcutBridge`?

Now let's look at who sets `keyboardShortcutBridge`. It seems there's a codepath where this object isn't set, so let's find out. When we load SwiftUI's binary in Hopper and search for this name, we find quite a few matches:

![Hopper search for keyboardShortcutBridge](/assets/img/2021/fixing-keyboardshortcut-in-swiftui/keyboardShortcutBridge.png)

Now let's analyze what we see here:

- There's a class named `KeyboardShortcutBridge` in SwiftUI
- It has one method marked `@objc`: `_performShortcutKeyCommand:`, therefore objc metadata is emitted (init, cxx_destruct)
- It uses `UIKeyCommand` under the hood, an API you will be familiar if you ever added keyboard support on iOS
- There is a setter for the Swift property that sets this object: `SwiftUI.UIHostingController.keyboardShortcutBridge.setter`

Using Xcode's breakpoint list isn't working too well for SwiftUI. Adding the setter there isn't working (fully qualified). Instead, let's use lldb directly and fuzzy-search for the breakpoint. You want to stop your program early (before the `UIHostingController` is created) and add the breakpoint manually:

```
(lldb) breakpoint set --func-regex keyboardShortcutBridge
Breakpoint 2: 3 locations.
```

Great! Now let's look at the 3 matches:

```
(lldb) breakpoint list
Current breakpoints:
1: file = '/Users/steipete/Projects/TempProjects/SwiftUIKeyboard/SwiftUIKeyboard/ViewController.swift', line = 22, exact_match = 0, locations = 1, resolved = 1, hit count = 1

  1.1: where = SwiftUIKeyboard`SwiftUIKeyboard.ViewController.viewDidAppear(Swift.Bool) -> () + 164 at ViewController.swift:22:20, address = 0x000000010115b444, resolved, hit count = 1 

2: regex = 'keyboardShortcutBridge', locations = 3, resolved = 3, hit count = 0
  2.1: where = SwiftUI`generic specialization <SwiftUI.ModifiedContent<SwiftUI.AnyView, SwiftUI.RootModifier>> of SwiftUI.UIHostingController.keyboardShortcutBridge.setter : Swift.Optional<SwiftUI.KeyboardShortcutBridge>, address = 0x00007fff57a376b0, resolved, hit count = 0 
  2.2: where = SwiftUI`SwiftUI.UIHostingController.keyboardShortcutBridge.getter : Swift.Optional<SwiftUI.KeyboardShortcutBridge>, address = 0x00007fff57a59960, resolved, hit count = 0 
  2.3: where = SwiftUI`SwiftUI.UIHostingController.keyboardShortcutBridge.setter : Swift.Optional<SwiftUI.KeyboardShortcutBridge>, address = 0x00007fff57a59990, resolved, hit count = 0 
```

That's good enough. And sure enough - the setter is never hit, in the non-working case. Once we apply the workaround, the method is hit:

![Xcode backtrace for keyboardShortcutBridge](/assets/img/2021/fixing-keyboardshortcut-in-swiftui/keyboardShortcutBridge-setter.png)

We see that the code that is responsible for calling the setter is in `didChangeAllowedBehaviors`. 

Next, let's see if there are any other places that would call this setter. I like to use a full pseudo-code export of SwiftUI. You can create this via Hopper -> File -> Produce Pseudo-Code File For All Procedures…. This will take many hours and produce a file named `SwiftUI.m` with over 100MB. Once this is done, use a text editor that can open such large files [^2] and search for `SwiftUI.UIHostingController.keyboardShortcutBridge.setter`. The only two code paths are these:

[^2]: In the early years, Sublime Text was my editor of choice for that, but nowadays the Electron-based Visual Studio Code is way faster in both opening and searching this file.

- `int _$s7SwiftUI19UIHostingControllerC25didChangeAllowedBehaviors4from2toyAC0gH0Vyx_G_AItF(int arg0)`
- `void _$s7SwiftUI19UIHostingControllerC25didChangeAllowedBehaviors4from2toyAC0gH0Vyx_G_AItFAA15ModifiedContentVyAA7AnyViewVAA12RootModifierVG_Tg5(int arg0)`
 
 This is mangled Swift, but it's not hard to see what the unmangled function name is called - it's our `didChangeAllowedBehaviors(from:to")` and a lambda inside it. Nowhere else. 
 
 ## Who triggers `didChangeAllowedBehaviors`?
 
 Who triggers an allowed behavior change? We can search for `SwiftUI.UIHostingController.allowedBehaviors.setter`, since `didChangeAllowedBehaviors` is triggered when the setter is invoked.
  
 - `_$s7SwiftUI16AppSceneDelegateC9sceneItemAA0D4ListV0G0VyF()`
 - `_$s7SwiftUI16RootViewDelegateC07hostingD0_9didMoveToyAA010_UIHostingD0CyxG_So8UIWindowCSgtAA0D0RzlF(int arg0, int arg1)`

So there's two mechanisms that trigger this:
- SwiftUI-based app lifecycle
- A root view delegate

This matches our previous tests. SwiftUI app lifecycle works, and if we add the `UIHostingController` as a root view controller, the RootView delegate also triggers the change. We can check via a fuzzy breakpoint if a `RootViewDelegate` is created in the non-working variant via `breakpoint set --func-regex RootViewDelegate`, and sure enough there are 13 matches, but none fires. 

When searching for `RootViewDelegate(` in the full-text `SwiftUI.m` file, there is only one match, in `s7SwiftUI14_UIHostingViewC15didMoveToWindowyyF`. This further confirms our theory. It seems Apple simply forgot a code path to create the keyboard shortcut bridge for the most likely use case - using SwiftUI in existing UIKit apps, where it makes sense. 

## Tweaking the Workaround

We can make the workaround slightly better and pack it into an extension. If we avoid making the temporary window key, we can skip a whole class of issues that appears when the key window is unexpectedly changed.

```swift
extension UIHostingController {

    /// Applies workaround so `keyboardShortcut` can be used via SwiftUI.
    ///
    /// When `UIHostingController` is used as a non-root controller with UIKit app lifecycle,
    /// keyboard shortcuts created in SwiftUI are not working (as of iOS 14.4).
    /// This workaround is harmless and triggers an internal state change that enables keyboard shortcut bridging.
    /// See https://steipete.com/posts/fixing-keyboardshortcut-in-swiftui/
    func applyKeyboardShortcutFix() {
        #if !targetEnvironment(macCatalyst)
        let window = UIWindow()
        window.rootViewController = self
        window.isHidden = false;
        #endif
    }
}
``` 

Because the window itself is shown and deallocated within the same runloop, it will never be visible. This workaround only uses public API and is safe to use. I have reported this issue to Apple via FB8984997.

[You can read the full bug report and sample project here.](https://github.com/PSPDFKit-labs/radar.apple.com/commit/8768d5c9fecd602625cc10b7a7c98f2bbc0cda4a)

## Conclusion

I hope this post is helping folks when they google "keyboardShortcut SwiftUI not working", provides a safe workaround and inspires a few folks to dig deeper. Swift is harder to reverse engineer, but it's still possible. This was the first time I had to set breakpoints for binary Swift symbols, so it's good to see that this still works when using lldb manually.