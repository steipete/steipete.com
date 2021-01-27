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

## Color Picker in SwiftUI

The `ColorPicker` view in SwiftUI is similar to UIKit's `UIColorWell` control. There's no native way to manually present the color picker, however it's trivial to bridge and present `UIColorPickerViewController` if needed.

```swift
struct ContentView: View {
    @State var color = Color.white

    var body: some View {
	    ColorPicker("Set color", selection: $color)
    }
}
```

For more advanced use cases, we need to look at UIKit.

## Using `UIColorPickerViewController` in UIKit

Apple's `UIColorPickerViewController` has a compact API and is straightforward to use. You can use the delegate-pattern to be notified about `selectedColor` property changes, or use KVO.

Using Combine's KVO wrapper is an extremely elegant way to receive color changes:

```swift
let colorPicker = UIColorPickerViewController()
cancellable = picker.publisher(for: \.selectedColor)
        .sink { color in
        print("New color set: \(color)")
        }
present(picker, animated: true, completion: nil)
```

While Apple also offers a `colorPickerViewControllerDidFinish` delegate call, this method isn't called when the picker is presented as a popover and dismissed by tapping outside the view - the call is only called when dismissing the control via the "Done" button when presented modally. Therefore, I question  the usefulness of this delegate.

## Presenting `UIColorPickerViewController`

While not documented, Apple designed the color picker to be presented modally. Everything works as expected. If we try to instead push the picker onto a navigation controller, the default behavior is pretty bad:

{% twitter https://twitter.com/steipete/status/1353718163612053504?s=21 %}

It looks like Apple hasn't tested this use case. The background color is missing, and there's a weird animation related to safeAreaInsets. The color picker is hosted as a remote view controller, which might explain some of these problems. Remote view controllers in UIKit are finicky and often have interesting bugs.

However, we can mitigate this somewhat if we embed `UIColorPickerViewController` into a custom container:

```swift
/// This is a wrapper to enable using UIKit's color picker via pushing in a navigation controller.
/// Presenting via modal presentation doesn't require a wrapper.
class ColorPickerWrapperController: UIViewController {

    /// There can only be one VC at all times; especially because Catalyst uses this with an external window.
    static let shared = UIColorPickerViewController()

    @objc let colorPicker = ColorPickerWrapperController.shared

    override init(nibName nibNameOrNil: String?, bundle nibBundleOrNil: Bundle?) {
        super.init(nibName: nibNameOrNil, bundle: nibBundleOrNil)

				// This is the minimum size the picker will work in.
        self.preferredContentSize = CGSize(width: 320, height: 500)
    }

    override func viewDidLoad() {
        super.viewDidLoad()

        add(childViewController: colorPicker)
        // Apple forgot defining a color for the picker
        view.backgroundColor = .white
    }

    required init?(coder: NSCoder) {
        fatalError("init(coder:) has not been implemented")
    }
}
```

Using this setup, the color picker can be pushed on a navigation controller stack. There's a short flickering as the remote plugin is loaded, but it's usable. These issues have been reported under FB8980868.

## Catalyst Is Different

In a surprising decision, Apple shows a completely different color picker when your app runs on the Mac, and it doesn't matter if the app runs via Catalyst (Scaled Interface), Catalyst (Optimized for Mac) or via the iOS Emulation that's new on Apple Silicon (Designed for iPad).

Especially the last mode is puzzling and will cause compatibility issues, as the color picker works completely different on the Mac.

The Mac version uses `NSColorPickerMatrixView`, an AppKit view, which is [bridged to UIKit via `_UINSView`](https://twitter.com/steipete/status/1353836791774777345).

{% twitter https://twitter.com/steipete/status/1353828767509188608?s=21 %}

The most important difference is that pressing the "Show Colors..." Button dismisses the picker and shows the default Mac color picker instead. The idea is that the color can be further tweaked using this window. This however only works if you ensure that the `UIColorPickerViewController` is kept around. I reported this surprising behavior as FB8981193 to Apple, and they indeed confirmed that the color picker controller must be kept around in order for that to work.

```swift
class ColorPickerSigleton {
    /// There can only be one VC at all times; especially because Catalyst uses this with an external window.
    static let shared = UIColorPickerViewController()
```

If you use `UIColorWell`, then this is automatically handled for you.

If the color picker is used via Catalyst's scaled mode, then this scaling [reduces the size of the picker to 0.77](https://twitter.com/steipete/status/1353708480511750148?s=21). This bug is reported via FB8980868. I recommend switching to the Optimized for Mac scaling mode to make the color picker normal sized. (Careful about [surprising crashes](https://steipete.com/posts/forbidden-controls-in-catalyst-mac-idiom/) when enabling this mode)

## Understanding AppKit's `NSColorPanel`

To understand why it's required to use `UIColorPickerViewController` like a singleton in Mac Catalyst, we need to look at AppKit. The `NSColorPanel` is designed to be a singleton and can only be displayed once per app:

```swift
	func applicationDidFinishLaunching(notification: NSNotification) {
		let colorPanel = NSColorPanel.sharedColorPanel()
		colorPanel.setTarget(self)
		colorPanel.setAction(Selector("colorDidChange:"))
		colorPanel.setAction.makeKeyAndOrderFront(self)
		colorPanel.setAction.continuous = true
	}

	func colorDidChange(sender: AnyObject) {
		if let colorPicker = sender as? NSColorPanel {
			print("New color: \(colorPicker.color\)")
		}
	}
```

## Conclusion

Apple's new color picker is a great addition to UIKit. It hasn't been widely tested and misses some polish, but it's a good and simple choice for selecting colors.