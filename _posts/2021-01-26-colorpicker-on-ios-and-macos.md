---
layout: post
title: "ColorPicker Under The Hood"
date: 2020-12-14 11:30:00 +0200
tags: iOS development
image: /assets/img/2021/apple-silicon-ci/trippin.png
description: "Let's look at the new color picker on iOS 14, Catalyst, and its AppKit legacy: NSColorPanel"
---

While macOS offers a system-provided color picker since OS X 10.0 Cheetah, iOS developers had to wait a bit longer. With iOS 14, Apple added [`UIColorPickerViewController`](https://developer.apple.com/documentation/uikit/uicolorpickerviewcontroller) and [`UIColorWell`](https://developer.apple.com/documentation/uikit/uicolorwell), which somewhat correspond to their older AppKit parents [`NSColorPanel`](https://developer.apple.com/documentation/appkit/nscolorpanel) and [`NSColorWell`](https://developer.apple.com/documentation/appkit/nscolorwell).

I've been taking a closer look at this control, and of course it's full or surprises.

## Using UIColorPickerViewController

The new iOS color picker can be either used directly, or is invoked automatically via the `UIColorWell` control. We'll focus on the manual use case.

## Color Picker in SwiftUI

```swift
struct ContentView: View {
    @State private var bgColor = Color.white

    var body: some View {
        VStack {
            ColorPicker("Set the background color", selection: $bgColor)
        }
        .frame(maxWidth: .infinity, maxHeight: .infinity)
        .background(bgColor)
    }
}
```

## Understanding AppKit's NSColorPanel

In AppKit, the NSColorPanel is shared over the whole app.

TODO: responder chain???

```
	func applicationDidFinishLaunching(aNotification: NSNotification) {
		let cp = NSColorPanel.sharedColorPanel()
		cp.setTarget(self)
		cp.setAction(Selector("colorDidChange:"))
		cp.makeKeyAndOrderFront(self)
		cp.continuous = true
	}

	func colorDidChange(sender:AnyObject) {
		if let cp = sender as? NSColorPanel {
			print(cp.color)
			self.window.backgroundColor = cp.color
		}
	}
```

## Conclusion