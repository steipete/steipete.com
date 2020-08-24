---
layout: post
title: "Logging in Swift"
date: 2020-08-24 19:00:00 +0200
tags: iOS development
image: /assets/img/2020/swift-logging/logd.jpeg
---

<style type="text/css">
div.post-content > img:first-child { display:none; }
</style>

With iOS 14, Apple greatly improved the built-in logging framework and added many missing pieces. Is OSLog now something that finally can be used?

## Why OSLog is Awesome

Every developer uses some form of logging, be it `NSLog` in Objective-C and `print` in Swift. These helpers are convenient, but do not scale - there's no concept to classify messages.

Apple [introduced os_log](https://developer.apple.com/documentation/os/os_log) with iOS 10 and macOS 10.12, in an attempt to provide a better, universal logging system[^7]. It supersedes the aging ASL (Apple System Logger) and comes with features expected from a modern logging system:

- **Categorization and filtering:** Log levels, grouping via subsystem and category.
- **Privacy:** Dynamic strings, collections and arrays are replaced to preserve personally identifiable information. This can be overridden on a per-parameter basis.
- **Usability:** The logging information collects calling info for you. Integrated into the system via Console.app and the log command line tool. If activity tracing is used, logs are automatically correlated.
- **Performance:** Logs are stored extremely efficient and all system log is in once place.

You might already stumbled over the log log, as you now need a 3rd breakpoint when investigating where a log messages is coming from: Next to `NSLog` and `CFLog` you now need a breakpoint to `_os_log_impl` as well.

[^7]: This post does not address `os_signpost`, `os_trace` and `os_activity` - all amazing features that fit into the logging concept and that you should be using.

## The New Swift Logging Framework

Instead of calling `os_log`, you can now use the new [`Logger`](https://developer.apple.com/documentation/os/logger) struct in Swift — at least if you have the luxury of exclusively supporting iOS 14 already.

```swift
let logger = Logger(subsystem: "com.steipete.LoggingTest", category: "main")
logger.info("Logging \(obj.description, privacy: .public)")
```

This innocent looking code uses a bunch of new tricks! First of, the new [String Interpolation feature of Swift 5](https://talk.objc.io/episodes/S01E143-string-interpolation-in-swift-5) is used to make it easy to customize the privacy of data.

Secondly, Apple modified the Swift compiler to allow [Compile Time Constant Expressions](https://gist.github.com/marcrasi/b0da27a45bb9925b3387b916e2797789) to evaluate the string at compile time - this ensure that logging is extremely fast. The technology behind is fascinating, and it seems that it was worth the wait[^6]. For more example code, see Apple's excellent documentation on [Generating Log Messages from Your Code](https://developer.apple.com/documentation/os/logging/generating_log_messages_from_your_code).

[^6]: Apple claimed in the [WWDC 2016 video about Unified Logging](https://developer.apple.com/videos/play/wwdc2016/721/) that support for Swift is coming "real soon". It took them 4 years, but they finally delivered on a worthy wrapper.

## Calling `os_log` in Swift

If you still need to use iOS 13 or are curious how things worked back in the days, here's how calling `os_log` works. Naively, you might try following approach:
```swift
os_log("foo: \(x) \(obj.description)", log: OSLog.default, type: .debug)
```

The compiler complains right away:

> Cannot convert value of type 'String' to expected argument type 'StaticString'

`os_log` requires the use of a Obj-C style format strings for performance and security reasons.[^4] At the same time Apple also strongly recommends to avoid wrapping os_log - so there's really no way around using following syntax:

```swift
os_log("foo: %@ %@", log: .default, type: .debug, x, obj.description)
```

[^4]: It's fairly easy to cause undefined behavior with NSLog, where Swift string interpolation and Swift format specifiers are mixed. There's a [great example on StackOverflow](https://stackoverflow.com/a/53025803/83160). `os_log` also supports a few special identifiers that are not part of NSString, so it's not exactly the same.

### What's the Problem with Wrapping OSLog?

OSLog is designed to accept static strings, in order to be fast and correctly separate between dynamic data (private) and static data (public). Most wrappers simply forward messages via `os_log("%{public}@"...` which makes all strings dynamic and public, removing most of the benefits of OSLog.

Apple maintains a separate [swift-log](https://github.com/apple/swift-log) logging framework for Swift on Server[^8]. The framework supports plugins, and there's a 3rd-party [swift-log-oslog plugin](https://github.com/chrisaljoudi/swift-log-oslog) which forward logs to the `os_log` machinery. Apple does link to it but also explicitly warns about usage:

[^8]: There are efforts underway to try to unify or at least better align the two logging frameworks, see the Swift Forum [Logging levels for Swift’s server-side logging APIs and new os_log APIs](https://forums.swift.org/t/logging-levels-for-swifts-server-side-logging-apis-and-new-os-log-apis/20365)

>Note: we recommend using os_log directly as described here. Using os_log through swift-log using this backend will be less efficient and will also prevent specifying the privacy of the message. The backend always uses %{public}@ as the format string and eagerly converts all string interpolations to strings. This has two drawbacks: 1. the static components of the string interpolation would be eagerly copied by the unified logging system, which will result in loss of performance. 2. It makes all messages public, which changes the default privacy policy of os_log, and doesn't allow specifying fine-grained privacy of sections of the message. In a separate on-going work, Swift APIs for os_log are being improved and made to align closely with swift-log APIs. References: Unifying Logging Levels, Making os_log accept string interpolations using compile-time interpretation.

There are also surprising bugs like [doubles that are not logged correctly](https://stackoverflow.com/questions/50937765/why-does-wrapping-os-log-cause-doubles-to-not-be-logged-correctly), which can be very hard to debug.

## Accessing Log Files

Log files are useful whenever users report a bug or a crash is being sent.  Most logging frameworks therefore include a rolling file logger which makes it easy to access logs on-demand.

Apple's [solution](https://developer.apple.com/forums/thread/650843?answerId=616460022) to accessing logs is either `sysdiagnose` or calling `log collect --device` on a terminal, possibly filter the output via `--start` or `--last` limits and thens ending the output manually. 

This might work if your audience is macOS developers, but if you're targeting regular users, it's much harder to make them follow a tutorial and then explain how to send a multiple-hundred-megabyte sysdiagnose. Furthermore, this will only work if your user runs a recent version of macOS.

In all practicality, this resulted in basically no app using `os_log` directly, which completely defeats the point. The most used logging frameworks to date seem to be [`SwiftyBeaver`](https://github.com/SwiftyBeaver/SwiftyBeaver) and [`CocoaLumberjack`](https://github.com/CocoaLumberjack/CocoaLumberjack).

This has been a point of personal frustration for me, there's [multiple](https://developer.apple.com/forums/thread/66984) forum entires where people are just baffled that this doesn't exist. I've brought this up every year except 2020 in the WWDC labs. The usual answer was "Apple cares about privacy so accessing logs is a no-go" — but the alternative is that people use other logging frameworks that have no privacy-preserving features built-in.

## New in iOS 14: OSLogStore

With [`OSLogStore`](https://developer.apple.com/documentation/oslog/oslogstore), Apple added API to access the log archive programmatically. It allows accessing `OSLogEntryLog` which contains all log info you possibly ever need. 

The new store access is available for all platform versions of 2020. It's the missing piece in the `os_log` story that finally will get us the best of both worlds. Let's look at how this works:

```swift
func getLogEntries() throws -> [OSLogEntryLog] {
    // Open the log store
    let logStore = try OSLogStore(scope: .currentProcessIdentifier)
    
    // Get all logs from the last hour
    let oneHourAgo = logStore.position(date: Date().addingTimeInterval(-3600))
    
    // Fetch log objects.
    let allEntries = try logStore.getEntries(at: oneHourAgo)
    
    // Filter log to be relevant for our specific subsystem
    // and remove other elements (signposts etc)
    return allEntries
        .compactMap { $0 as? OSLogEntryLog }
        .filter { $0.subsystem == subsystem }
}
```

The code is fairly straightforward, however above version has various issues. It works on macOS but doesn't work on iOS.

### Swift Overlay Issues

Apple forgot to add the Swift overlay shims on iOS, so we need to use some tricks to access the ObjC enumerator (FB8518476).

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

The next issue is that the entries enumerator has a built-in way to filter via the `predicate:`, however this doesn't seem to work (FB8518539).

```swift
// TODO: How to format the predicate? See "PREDICATE-BASED FILTERING" in `man log`.
// Things I tried unsuccessfully:
// - NSPredicate(format: "subsystem == %@", "com.steipete.LoggingTest")
// - NSPredicate(format: "subsystem == \"com.steipete.LoggingTest\"")
```

Credit for the syntax goes to Ole Begemann, wo has been investigating `OSLogStore` early on and maintains a [OSLogStoreTest](https://github.com/ole/OSLogStoreTest) sample project. 

### iOS Entitlement Issues

While it's fairly easy to work around the first two bugs, the third one is a real show-stopper. While accessing the log store works great on macOS (AppKit) and macOS (Catalyst), it fails with an entitlement error on iOS:

```
Error Domain=OSLogErrorDomain Code=9 "Client lacks entitlement to perform operation"
UserInfo={NSLocalizedDescription=Client lacks entitlement to perform operation, _OSLogErrorInternalCode=14}
```

It is not documented why this wouldn't work nor what entitlement is required. There's [anecdotical](https://twitter.com/bjtitus/status/1276211162506424323) [evidence](https://twitter.com/justkwin/status/1276271590360199172) on Twitter that Apple's intent is to allow this on iOS, however we're slowly nearing the end of Apple's beta cycle and there hasn't been an official announcement nor a fix.

If you feel strongly about this, please also report a radar and engage in the [discussion](https://developer.apple.com/forums/thread/658229) on Apple's Developer Forum.

### Digging Deeper

How does `OSLogStore` access logs in the first place? The store is part of the OSLog framework, which includes a small XPC service. At initialization time, the store opens a synchronous XPC request to `com.apple.OSLogService`, the service included in the framework.

`OSLogService` ensures logs are filtered for the current process and then accesses the `OSLogEventStore`. `OSLogEventStore` is implemented in the private `LoggingSuppport.framework`. Here, you can see that it connects to `logd` and also captures the failure condition which ultimately produces the entitlement error.

If we keep digging and also take a look at `logd`, we find references to various private entitlements:

- `com.apple.private.logging.admin`
- `com.apple.private.logging.diagnostic`

The admin one is required for accessing the log store. If we attach to both our app and logd, we can modify the flow at runtime to trick it into believing the entitlement is there, and voila, logging works on iOS!

{% twitter https://twitter.com/khaost/status/1297647313804857345 %}

Full credit goes to [Khaos Tian](https://twitter.com/khaost) who took the time to figure out the details. Attaching lldb to logd requires [SIP to be disabled](https://www.imore.com/how-turn-system-integrity-protection-macos), since logd is hardened and doesn't include the `get-task-allow` debug entitlement.

## Streaming OSLog

It is often desirable to stream log messages as they come in. This is a feature of many popular analytics and logging frameworks — of course OSLog also needs to this feature.

Lldb is doing exactly that, it listens to OSLog messages and prints the streams as they are created. Thankfully lldb is open source, so we can look at [DarwinLogCollector.cpp](https://github.com/llvm-mirror/lldb/blob/master/tools/debugserver/source/MacOSX/DarwinLog/DarwinLogCollector.cpp) which does exactly that[^1]. 

[^1]: A more convenient implementation is in [FLEX](https://github.com/Flipboard/FLEX/blob/master/Classes/GlobalStateExplorers/SystemLog/FLEXOSLogController.m).

The SPI[^2]: we need is in [ActivityStreamSPI.h](https://github.com/apple/llvm-project/blob/apple/master/lldb/tools/debugserver/source/MacOSX/DarwinLog/ActivityStreamSPI.h) which is part of lldb. (It contains various structs which cannot be redefined in Swift, this file needs to be plain C) The main functions that are important to us are:

[^2]: System Programming Interface. This is private API that's used within Apple's frameworks, thus it's extremely rare that SPI is changed, unlike completely internal private API.

```c
// Get an activity stream for a process.
typedef os_activity_stream_t (*os_activity_stream_for_pid_t)(
    pid_t pid, os_activity_stream_flag_t flags,
    os_activity_stream_block_t stream_block);

// Start the activity stream.
typedef void (*os_activity_stream_resume_t)(os_activity_stream_t stream);

// Stop the activity stream.
typedef void (*os_activity_stream_cancel_t)(os_activity_stream_t stream);

// Get a formatted message out of an activity.
typedef char *(*os_log_copy_formatted_message_t)(os_log_message_t log_message);
```

Given these functions we can write a Swift[^3] class that accesses the streaming log. You can see a reference implementation in [my gist of `OSLogStream.swift`](https://gist.github.com/steipete/459a065f905a41f8f577fb02ef34206e). While this works, SPI is still private API and you shouldn't ship this in the App Store (FB8519418).

<script src="https://gist.github.com/steipete/459a065f905a41f8f577fb02ef34206e.js"></script>

[^3]: There's currently no way to correctly define C structs in Swift, so importing the C file `ActivityStreamSPI.h` is a requirement. This is a shame, as it makes releasing that as SwiftPM module nearly impossible. We could rewrite everything in Objective-C to make it work, or hope that mixed-mode projects will be supported eventually.

## Conclusion

OSLog is an extremely awesome API and all the pieces are there - just currently locked away behind private API, entitlements and some bugs. It remains exciting if Apple will fix the various issues in `OSLogStore` before the GM. Or does it work as intended?

Did I miss something? [Hit me up on Twitter](https://twitter.com/steipete) or [open a Pull Request](https://github.com/steipete/steipete.com) to help with typos. Thanks! 🙏

## Further Reading

- [Unified Logging and Activity Tracing, WWDC 2016](https://developer.apple.com/videos/play/wwdc2016/721/)
- [Explore logging in Swift, WWDC 2020](https://developer.apple.com/videos/play/wwdc2020/10168/)
- [Making `os_log` public on macOS Catalina](https://saagarjha.com/blog/2019/09/29/making-os-log-public-on-macos-catalina/)
- [Generating Log Messages from Your Code](https://developer.apple.com/documentation/os/logging/generating_log_messages_from_your_code)
- [Meet the new Logger API](https://wwdcbysundell.com/2020/meet-the-new-logger-api/)