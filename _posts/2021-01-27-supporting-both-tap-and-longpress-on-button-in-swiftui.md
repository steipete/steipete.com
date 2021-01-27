---
layout: post
title: "Supporting both Tap and LongPress on a Button in SwiftUI"
date: 2021-01-27 18:30:00 +0200
tags: iOS SwiftUI development
image: /assets/img/2021/tap-longpress-button-swiftui/header.gif
description: "My task today was quite simple: adding an optional long-press handler to a button in SwiftUI. Not so difficult, eh? You'd be surprised how tricky it can get."
---

My task today was quite simple: adding an optional long-press handler to a button in SwiftUI. A regular tap opens our website and a long press does… something else. Not so difficult, right?

## Naive First Version

Here's my first naive iteration:

```swift
Button(action: {
    openWebsite(.pspdfkit)
}) {
    Image("pspdfkit-powered")
        .renderingMode(.template)
        .onLongPressGesture(minimumDuration: 2) {
            print("Secret Long Press Action!")
        }
}
```

While above works to detect a long press, by adding a gesture to the image, the button no longer fires. Alright, not quite what we want.  Let's move the gesture out of the label and to the button.

## Moving Things Around-Version

```swift
Button(action: {
    openWebsite(.pspdfkit)
}) {
    Image("pspdfkit-powered")
        .renderingMode(.template)
}
.onLongPressGesture(minimumDuration: 2) {
    print("Secret Long Press Action!")
}
```

Great! Now the button tap works again - unfortunately the long press gesture doesn't work anymore. Okay, let's use `simultaneousGesture` to tell SwiftUI that we really care about both gestures.

## Being Fancy with `simultaneousGesture`

```swift
Button(action: {
    openWebsite(.pspdfkit)
}) {
    Image("pspdfkit-powered")
        .renderingMode(.template)
}
.simultaneousGesture(LongPressGesture().onEnded { _ in
    print("Secret Long Press Action!")
})
Spacer()
```

Great - that works. However now we always trigger both the long press and the action, which isn't quite what we want. We want either-or, so let's try adding a second gesture instead:

## Two Gestures Are Better Than One

```swift
Button(action: {
	// ignore
}) {
    Image("pspdfkit-powered")
        .renderingMode(.template)
}
.simultaneousGesture(LongPressGesture().onEnded { _ in
    print("Secret Long Press Action!")
})
.simultaneousGesture(TapGesture().onEnded {
    print("Boring regular tap")
    openWebsite(.pspdfkit)
})
Spacer()
```

It… works! It does exactly what we expect and it's nicely calling either tap or long press. Wohoo! So let's do some QA and test everywhere. iOS 13: check. iOS 14: check. Let's compile the Catalyst version to be sure. And: it does not work. Neither tap nor long tap. The button has no effect at all.

## Catalyst… always Catalyst!

If we can ignore the long press on Catalyst, then this combination works at least for the regular action.

```swift
    @State var didLongPress = false

    var body: some View {
        Button(action: {
            if didLongPress {
                didLongPress = false
            } else {
                print("Boring regular tap")
                openWebsite(.pspdfkit)
            }
        }) {
            Image("pspdfkit-powered")
                .renderingMode(.template)
        }
        // None of this ever fires on Mac Catalyst :(
        .simultaneousGesture(LongPressGesture().onEnded { _ in
            didLongPress = true
            print("Secret Long Press Action!")
        })
        .simultaneousGesture(TapGesture().onEnded {
            didLongPress = false
        })
    }
```

In our case, we really want the long press though, so what to do? I remembered a trick I used im my [Presenting Popovers from SwiftUI](https://pspdfkit.com/blog/2020/popovers-from-swiftui-uibarbutton/) article, we can use a `ZStack` and just use UIKit for what doesn't work in SwiftUI.

## The Nuclear Option

The usage is simple:

```swift
LongPressButton(action: {
    openWebsite(.pspdfkit)
}, longPressAction: {
    print("Secret Long Press Action!")
}, label: {
    Image("pspdfkit-powered")
        .renderingMode(.template)
})
```

Now, let's talk about this `LongPressButton` subclass…

```swift
struct LongPressButton<Label>: View where Label: View {
    let label: (() -> Label)
    let action: () -> Void
    let longPressAction: () -> Void

    init(action: @escaping () -> Void, longPressAction: @escaping () -> Void, label: @escaping () -> Label) {
        self.label = label
        self.action = action
        self.longPressAction = longPressAction
    }

    var body: some View {
        Button(action: {
        }, label: {
            ZStack {
                label()
                // Using .simultaneousGesture(LongPressGesture().onEnded { _ in works on iOS but fails on Catalyst
                TappableView(action: action, longPressAction: longPressAction)
            }
        })
    }
}

private struct TappableView: UIViewRepresentable {
    let action: () -> Void
    let longPressAction: () -> Void

    typealias UIViewType = UIView

    func makeCoordinator() -> TappableView.Coordinator {
        Coordinator(action: action, longPressAction: longPressAction)
    }

    func makeUIView(context: Self.Context) -> UIView {
        UIView().then {
            let tapGestureRecognizer = UITapGestureRecognizer(target: context.coordinator,
                                                              action: #selector(Coordinator.handleTap(sender:)))
            $0.addGestureRecognizer(tapGestureRecognizer)
            let doubleTapGestureRecognizer = UILongPressGestureRecognizer(target: context.coordinator,
                                                                          action: #selector(Coordinator.handleLongPress(sender:)))
            doubleTapGestureRecognizer.minimumPressDuration = 2
            doubleTapGestureRecognizer.require(toFail: tapGestureRecognizer)
            $0.addGestureRecognizer(doubleTapGestureRecognizer)
        }
    }

    func updateUIView(_ uiView: UIView, context: Self.Context) { }

    class Coordinator {
        let action: () -> Void
        let longPressAction: () -> Void

        init(action: @escaping () -> Void, longPressAction: @escaping () -> Void) {
            self.action = action
            self.longPressAction = longPressAction
        }

        @objc func handleTap(sender: UITapGestureRecognizer) {
            guard sender.state == .ended else { return }
            action()
        }

        @objc func handleLongPress(sender: UILongPressGestureRecognizer) {
            guard sender.state == .began else { return }
            longPressAction()
        }
    }
}
```

And here we go. This version works exactly as we expect, on iOS 13, iOS 14, Catalyst on Catalina and Big Sur. **UIKit is verbose but it works.** And with the power of SwiftUI we can hide all that code behind a convenient new button subclass.

## Conclusion

So what's really about that secret long press action? It does enable the Debug Mode of [PDF Viewer](https://pdfviewer.io), showing various settings that aren't really useful for regular folks, but help with QA testing. If you're curious, download our app (it's free), long-press on our icon in the Settings footer and see.