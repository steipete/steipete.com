---
layout: post
title: "Logging in Swift"
date: 2020-08-05 12:00:00 +0200
tags: iOS development
image: 
---

With iOS 14, Apple greatly improved the built-in logging framework and seemingly added many missing pieces[^6]. Is OSLog now something that finally can be used?

[^6]: Apple claimed in the [WWDC 2016 video about Unified Logging](https://developer.apple.com/videos/play/wwdc2016/721/) that support for Swift is coming "real soon". It took them 4 years, but they finally delivered on a worthy wrapper.

What do you mean, not usable? We use `os_log` for years now!

## Calling os_log in Swift

If you would naively try to use `os_log` in Swift like this:
```
os_log("foo: \(x) \(y.description)", log: OSLog.default, type: .debug)
```

The compiler complains right away:

> Cannot convert value of type 'String' to expected argument type 'StaticString'

`os_log` requires the use of a Obj-C style format strings for performance and security reasons.[^4] At the same time Apple also strongly recommends to avoid wrapping os_log.

[^4]: It's fairly easy to cause undefined behavior with NSLog, where Swift string interpolation and Swift format specifiers are mixed. There's a [great example on StackOverflow](https://stackoverflow.com/a/53025803/83160).

### Why Is Wrapping OSLog Bad?

The main issue with Apple's `os_log` is that its design only makes sense when you use it to directly construct a string. Strings are placed in a separate section of the binary, which makes os_log extremely fast and privacy preserving. Dynamic elements in the string construction are automatically masked.

This however implies that `os_log` should not be wrapped in another logging abstraction. [Apple warns for this in their swift-log project](https://github.com/apple/swift-log), which links to a 3rd-party [swift-log-oslog plugin](https://github.com/chrisaljoudi/swift-log-oslog), with the following warning:

>Note: we recommend using os_log directly as decribed here. Using os_log through swift-log using this backend will be less efficient and will also prevent specifying the privacy of the message. The backend always uses %{public}@ as the format string and eagerly converts all string interpolations to strings. This has two drawbacks: 1. the static components of the string interpolation would be eagerly copied by the unified logging system, which will result in loss of performance. 2. It makes all messages public, which changes the default privacy policy of os_log, and doesn't allow specifying fine-grained privacy of sections of the message. In a separate on-going work, Swift APIs for os_log are being improved and made to align closely with swift-log APIs. References: Unifying Logging Levels, Making os_log accept string interpolations using compile-time interpretation.

The wrapper uses `os_log("%{public}@"...` which makes all strings dynamic and public, removing most of the benefits of OSLog.

The other issue is that apps often include logs with a bug or crash report. Most logging frameworks include a rolling file logger which makes it easy to access logs on-demand. [Apple's recommended way to access logs](https://developer.apple.com/forums/thread/650843?answerId=616460022) of a user or tester is to ask them to call  `log collect --device` on a terminal, possibly filter the output via `--start` or `--last` limits and thens ending the output manually. 

>a tester with access to a Mac can use log collect --device as seen in the talk to get a logarchive, even if they do not have Xcode. Using --start or --last limits the time range of the logs, which can limit the size of the resulting logarchive.

- [Apple System Engineer](https://developer.apple.com/forums/thread/650843?answerId=616460022)

How many of your users are willing and able to do this? Furthermore, It will also only work if the user's iPhone or iPad is synced to macOS, and they have the latest stable version.

In all practicality, this resulted in basically no app using `os_log` directly, which completely defeats the point.

## Why OSLog is Awesome

Every developer uses some form of logging, be it `NSLog` in Objective-C and `print` in Swift. These helpers are convenient, but do not scale - there's no concept to classify messages.

Apple [introduced os_log](https://developer.apple.com/documentation/os/os_log) with iOS 10 and macOS 10.12, in an attempt to provide a better, universal logging system. It supersedes the aging ASL (Apple System Logger) and comes with features expected from a modern logging system:

- Log Levels (error, debug, info)
- Group messages inside subsystems and categories
- Extremely high performance - no need to use conditionals for logging.
- Privacy built-in. 
- Integrated into the system via Console.app and the log command line tool
- If activity tracing is used, logs are automatically correlated (TODO: how does that look?)

Most developers probably noticed (And that you now need a breakpoint to `_os_log_impl` next to `NSLog` and `CFLog` to find which component is responsible for a log message in the console)

This post does not address `os_signpost`, `os_trace` and `os_activity` - all amazing features that fit into the logging concept.

## New in iOS 14: OSLogStore

With [`OSLogStore`](https://developer.apple.com/documentation/oslog/oslogstore), Apple added API to access the log archive programmatically. The new store access is available for all platform versions of 2020.

It's the missing piece in the `os_log` story that finally will get us the best of both worlds. Let's look at how this works:

```swift
func getLogEntries() throws -> [OSLogEntryLog] {
    let logStore = try OSLogStore(scope: .currentProcessIdentifier)
    let oneHourAgo = logStore.position(date: Date().addingTimeInterval(-3600))
    let allEntries = try logStore.getEntries(at: oneHourAgo)
    return allEntries
        .compactMap { $0 as? OSLogEntryLog }
        .filter { $0.subsystem == subsystem }
}
```

`OSLogEntryLog` is a class that contains all log info your heart desires, from log level to message to thread to process.

### Swift Overlay Issues
However, above code has a few problems. It works on macOS but doesn't work on iOS. Apple forgot to add the Swift overlay shims there, so we need to use some tricks to access the ObjC enumerator (FB8518476).

```swift
#if os(macOS)
    let allEntries = try logStore.getEntries(at: oneHourAgo)
#else
    // FB8518476: The Swift shims for for the entries enumerator are missing.
    let allEntries = try Array(logStore.__entriesEnumerator(position: oneHourAgo, predicate: nil))
#endif
```

Since the non-sugared version also works on iOS, it's not necessary to do an if/else dance at all, but it's a good pattern to document how it should work, in the hopes that the bug gets fixed before the GM.

### Predicate Issues
The next issue is that the entries enumerator has a built-in way to filter via the `predicate:`, however none of my attempts were successful (FB8518539).

```swift
// TODO: How to format the predicate? See "PREDICATE-BASED FILTERING" in `man log`.
// Things I tried unsuccessfully:
// - NSPredicate(format: "subsystem == %@", "com.steipete.LoggingTest")
// - NSPredicate(format: "subsystem == \"com.steipete.LoggingTest\"")
```

Credit for the syntax go to Ole Begemann, wo's been investigating OSLogStore early on and maintains a [OSLogStoreTest](https://github.com/ole/OSLogStoreTest) sample project. 

### iOS Entitlement Issues

While it's fairly easy to work around the first two bugs, the third one is a real show-stopper. While accessing the log store works great on macOS (AppKit) and macOS (Catalyst), it fails with an entitlement error on iOS:

```
Error Domain=OSLogErrorDomain Code=9 "Client lacks entitlement to perform operation" UserInfo={NSLocalizedDescription=Client lacks entitlement to perform operation, _OSLogErrorInternalCode=14}
```

It is not documented why this wouldn't work nor what entitlement is required. There's [anecdotical](https://twitter.com/bjtitus/status/1276211162506424323) [evidence](https://twitter.com/justkwin/status/1276271590360199172) on Twitter that Apple's intent is to allow this on iOS, however we're slowly nearing the end of Apple's beta cycle and there hasn't been an official announcement nor a fix.

#### Digging Deeper

How does `OSLogStore` access logs in the first place? The store is part of the OSLog framework, which includes a small XPC service. At initialization time, the store opens a synchronous XPC request to `com.apple.OSLogService`, the service included in the framework.

OSLogService ensures logs are filtered for the current process and then accesses the `OSLogEventStore`.

OSLogEventStore is implemented in the private `LoggingSuppport.framework`. Here, you can see that it connects to `logd` and also captures the failure condition which ultimately produces the entitlement error.

If we keep digging and also take a look at `logd`, we find references to various private entitlements:

- com.apple.private.logging.admin
- com.apple.private.logging.diagnostic

The admin one is required for accessing the log store. If we attach to both our app and logd, we can modify the flow at runtime to trick it into believing the entitlement is there, and voila, logging works on iOS!

<% twitter https://twitter.com/khaost/status/1297647313804857345 %>

Full credit goes to [Khaos Tian](https://twitter.com/khaost) who took the time to figure out the details. Attaching lldb to logd requires SIP disabled, since logd is hardened and doesn't include the `get-task-allow` debug entitlement.

## Streaming OSLog

It is often desirable to stream log messages as they come in. This is a feature of many popular analytics and logging frameworks, from Google Firebase to SwiftyBeaver.

If we look at Xcode, lldb is doing exactly the same - streaming log messages as our app is running. Thankfully lldb is open source, so we can look at [DarwinLogCollector.cpp](https://github.com/llvm-mirror/lldb/blob/master/tools/debugserver/source/MacOSX/DarwinLog/DarwinLogCollector.cpp) which does exactly that[^1]. 

[^1]: A more convenient implementation is in [FLEX](https://github.com/Flipboard/FLEX/blob/master/Classes/GlobalStateExplorers/SystemLog/FLEXOSLogController.m).

The SPI[^2]: we need is in [ActivityStreamSPI.h](https://github.com/apple/llvm-project/blob/apple/master/lldb/tools/debugserver/source/MacOSX/DarwinLog/ActivityStreamSPI.h) which is part of lldb. (It contains various structs which cannot be redefined in Swift, this file needs to be plain C)

[^2]: System Programming Interface. This is private API that's used within Apple's frameworks, thus it's extremely rare that SPI is changed, unlike completely internal private API.

Given that, we can write a Swift class that accesses the streaming log:

<script src="https://gist.github.com/steipete/459a065f905a41f8f577fb02ef34206e.js"></script>

Note: There's currently no way to correctly define C structs in Swift, so importing the C file `ActivityStreamSPI.h` is a requirement.[^3] 

[^3]: This is a shame, as it makes releasing that as SwiftPM module nearly impossible. We could rewrite everything in Objective-C to make it work, or hope that mixed-mode projects will be supported eventually.

While this works, SPI is still private API and you shouldn't ship this in the App Store (FB8519418).

## Conclusion

It remains exciting if Apple will fix the various issues in OSLogStore before the GM. OSLog is an extremely awesome API and all the pieces are there - just currently locked away behind private API, entitlements and some bugs.

## Further Reading

- [Unified Logging and Activity Tracing, WWDC 2016](https://developer.apple.com/videos/play/wwdc2016/721/)
- [Explore logging in Swift, WWDC 2020](https://developer.apple.com/videos/play/wwdc2020/10168/)
- [Making `os_log` public on macOS Catalina](https://saagarjha.com/blog/2019/09/29/making-os-log-public-on-macos-catalina/)