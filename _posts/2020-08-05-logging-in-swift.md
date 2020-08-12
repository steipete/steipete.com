---
layout: post
title: "Logging in Swift"
date: 2020-08-05 12:00:00 +0200
tags: iOS development
image: 
---

For the last few weeks I've been working on a side-project app to learn SwiftUI, Combine and [modern architecture patterns](https://github.com/pointfreeco/swift-composable-architecture). As the project grew, a logging system become more and more important.

## Why Invest in Logging?

Every developer uses some form of logging, be it `NSLog` in Objective-C and `print` in Swift. These helpers are convenient, but do not scale - there's no concept to classify messages.

## os_log

Apple [introduced os_log](https://developer.apple.com/documentation/os/os_log) with iOS 10 and macOS 10.12, which supersedes the aging ASL (Apple System Logger) and comes with features expected from a modern logging system:

- Log Levels (error, debug, info)
- Group messages inside subsystems and categories
- Extremely high performance - no need to use conditionals for logging.
- Privacy built-in.
- Integrated into the system via Console.app and the log command line tool
- If activity tracing is used, logs are automatically correlated (TODO: how does that look?)

Most developers probably noticed (And that you now need a breakpoint to `_os_log_impl` next to `NSLog` and `CFLog` to find which component is responsible for a log message in the console)

TODO: os_signpost/os_trace/os_activity

## Alternatives

- SwiftyBeaver
- CocoaLumberjack?

## Apple SwiftLog

https://github.com/apple/swift-log

## The new Apple Logging Framework

https://developer.apple.com/documentation/os/logging

TODO: How does os_activity and os_log play together?
https://developer.apple.com/documentation/os/logging/collecting_log_messages_in_activities

## OSLogStore

OSLogStore defines [a set of entries from the unified logging system](https://developer.apple.com/documentation/oslog/oslogstore) - 

https://github.com/ole/OSLogStoreTest
https://github.com/ole/OSLogStoreTest/blob/main/Shared/OSLogStorePrinter.swift

## Streaming os_activity via private API

https://github.com/limneos/oslog/blob/master/main.mm

lldb: lldb/trunk/tools/debugserver/source/MacOSX/DarwinLog/DarwinLogCollector.cpp
https://reviews.llvm.org/D66566

PrivateFrameworks/LoggingSupport.framework

https://github.com/Flipboard/FLEX/blob/master/Classes/GlobalStateExplorers/SystemLog/FLEXOSLogController.m

## Conclusion