---
layout: post
title:  "How to fix lldb: Couldn't IRGen expression"
date:   2020-06-03 20:00:00 +0200
tags: iOS development
---

A few weeks back some support tickets have been coming up with reports that people can't use the `lldb` debugger anymore after integrating [PSPDFKit](http://pspdfkit.com/). Instead of printing an object, they get `Couldn't IRGen expression, no additional error`. That's obviously not great, so I went down a rabbit hole to understand what's up here. 

## Analysis

The debugger from Xcode 11.5 (11E608c) fails to print information about variables using the `po`, `p` or `e` commands. 
Example command: `po window`
Result: `error: Couldn't IRGen expression, no additional error`

![](/assets/img/2020/lldb-debugging/xcode-lldb.png)

Why didn't we see this before? All our examples work, as they use the new `xcframework` format - and for some reason, everything works here.

Let's see what works and what doesn't:

- ✅ Creating an example with `xcframework` (this is the format we distribute) where our SDK is built with `BUILD_LIBRARY_FOR_DISTRIBUTION` enabled
- ✅ Mixed Obj-C/Swift Example via `framework`, `xcframework`, CocoaPods or Carthage. (Independent of `BUILD_LIBRARY_FOR_DISTRIBUTION`)
- ❌ Creating an example with `xcframework` without build for distribution
- ❌ Creating a custom example with `framework`
- ❌ Swift-only Example via CocoaPods or Carthage
- ❌ Swift-only Example via CocoaPods using `xcframework`

⚠️ Testing here is tricky - Apple saves absolute paths in the binary, so if you happen to have the same username on the build machine and your test machine, it might work, but fails somewhere else. It also seems that lldb uses the shared module cache. I ended up creating a fresh virtual machine with a generic username with snapshots, to ensure correct reproducibility. We also enabled `BUILD_LIBRARY_FOR_DISTRIBUTION` between the current release (9.3.3) and the upcoming version (9.4), which again changed the example results.

In mixed-mode projects, debugging works, but lldb complains:

```
(lldb) po window
warning: Swift error in fallback scratch context: error: failed to load module 'PSPDFSimple'


note: This error message is displayed only once. If the error displayed above is due to conflicting search paths to Clang modules in different images of the debugged executable, this can slow down debugging of Swift code significantly, since a fresh Swift context has to be created every time a conflict is encountered.

<UIWindow: 0x7fbd5d59a160; frame = (0 0; 375 667); hidden = YES; gestureRecognizers = <NSArray: 0x600001ad5dd0>; layer = <UIWindowLayer: 0x600001565ac0>>
```

![](/assets/img/2020/lldb-debugging/xcode-lldb-mixed.png)

We found [a workaround on StackOverflow](https://stackoverflow.com/questions/54776459/whats-the-solution-for-error-couldnt-irgen-expression-no-additional-error/61824142#61824142) that works around this issue by adding one Objective-C class and a bridging header to your project. ([KB Article](https://pspdfkit.com/guides/ios/current/knowledge-base/debugging-issues/))

We reported this bug as FB7718242 to Apple.

## Understanding the Issue

The workaround is acceptable, but we wanted to better understand what the issue here is - and be prepared in case this workaround stops working. Debugging is essential, so this is worth understanding properly.

On the linked StackOverflow workaround, there's an interesting comment:

>There is currently a hard requirement that the version of the swift compiler you use to build your source and the version of lldb you use to debug it must come from the same toolchain. Currently the swift debug information for types is just a serialization of internal swift compiler data structures. It also depends on local path information, which makes it hard to move around. There is a longer term effort to change that design, but for now you have to rebuild all your binaries every time you update your tools, and you can't use pre-built binaries. — [Jim Ingham, Apple Engineer](https://stackoverflow.com/questions/54776459/whats-the-solution-for-error-couldnt-irgen-expression-no-additional-error/61824142#61824142)

We found [SR-12783](https://bugs.swift.org/browse/SR-12783) which exactly explains the problem, and is also marked as resolved. The OP used the binary Instabug SDK, which is partially written in Swift. A very similar situation to ours.

> The binary Swift module encode a hardcoded path to a yaml file that only exists on the original developer's machine. You should let them know that they need to compile their binary framework with -no-serialize-debugging-options if they are planning to distribute them to another machine.

> LLDB has an embedded Swift compiler that will attempt to load the `.swiftmodule` for each Swift module in your program. The binary `.swiftmodule` is embedded in the .dSYM bundle for LLDB to find. When -serialize-debugging-options` is enabled the Swift compiler will serialize all Clang options (such as the `-ivfsoverlay` option added by Xcode's build system to find `all-product-headers.yaml`). This works really nice on the machine that the swift module was built on, but obviously isn't portable. We are working on lifting this dependency in LLDB, but that is still in progress.

> The `-no-serialize-debugging-options` option will omit those clang flags. The price for this is that you may need to pass one or two missing Clang options to LLDB manually via settings set target.swift-extra-clang-flags when you are debugging the framework itself now, but you may also get lucky and LLDB can piece together the necessary Clang flags from the main program. — [Adrian Prantl, Apple Compiler Engineer](https://bugs.swift.org/browse/SR-12783?focusedCommentId=56548&page=com.atlassian.jira.plugin.system.issuetabpanels%3Acomment-tabpanel#comment-56548)

# Trying Swift Trunk Snapshot

Adrian subsequently landed a [patch in lldb modifying `SwiftExpressionParser.cpp`](https://github.com/apple/llvm-project/pull/1220) to print the root cause of this failure. I went ahead and did try this with the [Swift Trunk Snapshot](https://swift.org/download/#snapshots). With the one from May 26, the error now looks different:

```
warning: Swift error in fallback scratch context: error: module 'Swift' was created for incompatible target x86_64-apple-ios13.0-simulator: /Users/steipete/Library/Developer/Xcode/DerivedData/ModuleCache.noindex/Swift-3K8REJ00QGV2U.swiftmodule

note: This error message is displayed only once. If the error displayed above is due to conflicting search paths to Clang modules in different images of the debugged executable, this can slow down debugging of Swift code significantly, since a fresh Swift context has to be created every time a conflict is encountered.

Can't construct shared Swift state for this process after repeated attempts.
Giving up.  Fatal errors:
error: module 'Swift' was created for incompatible target x86_64-apple-ios13.0-simulator: /Users/steipete/Library/Developer/Xcode/DerivedData/ModuleCache.noindex/Swift-3K8REJ00QGV2U.swiftmodule
```

![](/assets/img/2020/lldb-debugging/xcode-lldb-newtoolchain.png)

This is interesting, since we compile our SDK with [`BUILD_LIBRARY_FOR_DISTRIBUTION = YES`](https://swift.org/blog/library-evolution/), so with library evolution enabled. The incompatible target issue seems to be a different problem.

## Enabling lldb Logging 

There's [another trick](https://forums.swift.org/t/swiftpm-and-lldb/26966) in lldb that enables a great deal of logging: put `log enable lldb types` into `~/.lldbinit` and you see a great deal of logging in lldb.

When we use this in our original failing setup, we get more information, specifically the `all-product-headers.yaml` is listed that was also mentioned in [SR-12783](https://bugs.swift.org/browse/SR-12783).

```
SwiftASTContext("PSPDFKitUI")::GetASTContext() -- failed to initialize ClangImporter: error: virtual filesystem overlay file '/Users/steipete/Projects/lldb-debug-test/PSPDFKit/iOS/build/PSPDFKitUI/production/Build/Intermediates.noindex/ArchiveIntermediates/PSPDFKitUI.framework/IntermediateBuildFilesPath/PSPDFKitUI.build/Release-iphonesimulator/PSPDFKitUI.framework.build/all-product-headers.yaml' not found
```

## Comparing Binary Files

I was curious what `-no-serialize-debugging-options` changes in the binary, yso I tried using[Araxis Merge](https://www.araxis.com/merge/documentation-os-x/comparing-binary-files.en) to track the changes. Other useful tools are [Hex Fiend](https://ridiculousfish.com/hexfiend/) and [MachO-Explorer](https://github.com/DeVaukz/MachO-Explorer).

![](/assets/img/2020/lldb-debugging/araxis-merge.png)

That turned out to be another rabbit hole, since Xcode doesn't produce deterministic builds. Other build systems like Buck [use various workarounds](https://milen.me/writings/apple-linker-ld64-deterministic-builds-oso-prefix/) to achieve this, and there's an effort to enable [deterministic builds with clang and lld](http://blog.llvm.org/2019/11/deterministic-builds-with-clang-and-lld.html), but we're not yet there. 

We can compare the sizes of the Mach-O sections in the binary: 

`otool -l /bin/ls | grep -e '  cmd' -e datasize | sed 's/^ *//g'`

## SWIFT_SERIALIZE_DEBUGGING_OPTIONS

There's very little on the internet that even [documents](https://github.com/apple/swift-package-manager/pull/2713)[`SWIFT_SERIALIZE_DEBUGGING_OPTIONS`](https://twitter.com/evandcoleman/status/1266414571180429312), but it seems to be the same as `OTHER_SWIFT_FLAGS = -Xfrontend -no-serialize-debugging-options`. Luckily Swift is open source, so we can look up [the very commit introducing -no-serialize-debugging-options](https://github.com/apple/swift/commit/8ee17a4d0d0bba46a0b3b6e200c95d40a548a02e).

The problem: Setting the flag doesn't change anything for us. Here's the lldb logs for different scenarios:

- ✅ [mixed-mode debug with lldb and -no-serialize-debugging-options](https://gist.github.com/steipete/fb86213fbc7407d6c217277ee2be7ac1)
- ❌ [Swift-only debug with lldb and -no-serialize-debugging-options](https://gist.github.com/steipete/9eaaa17f552aef875e139a6e2fb9503f)
- ❌ [Swift-only debug with lldb and -no-serialize-debugging-options, latest toolchain for app](https://gist.github.com/steipete/9eaaa17f552aef875e139a6e2fb9503f)

## SR-12932

I filed [SR-12932 - Custom toolchain picks up wrong target based on iOS deployment target](https://bugs.swift.org/browse/SR-12932) and noticed that this can be worked around by raising the iOS Deployment Target to iOS 13.



This will need further investigation.