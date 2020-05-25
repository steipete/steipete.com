---
layout: post
title:  "null-character in strings & tales of Apple Radar"
date:   2020-05-24 10:00:00 +0200
tags: development iOS
---

We're finally dropping iOS 11 support the PSPDFKit SDK. While there's nothing major in terms of new features for iOS 12, this does help us to remove old workarounds for system-level bugs. Apple didn't drop any devices with the iOS 12 release, so everybody running iOS 11 can upgrade to iOS 12. (While there are no official statistics for iOS 11 usage, it seems to be already [below one percent](https://david-smith.org/iosversionstats/).)

# Our Process

Whenever we drop an iOS release, we take time to carefully modernize the codebase, remove deprecations and adopt new API. This is important to keep technical debt low and ensure that the codebase stays a comfortable place to work in.

The first step to remove iOS 11 is running a full-text search for "iOS 11" on the codebase. We document version-specific issues and I found something I've long forgot about:

```objc
/**
 Returns a string that has been cleaned of '\0' characters.
 Guarantees to return a copy.

 iOS 11 made it easy to create them: https://github.com/PSPDFKit/PSPDFKit/issues/12280
 rdar://34651564 http://openradar.appspot.com/34651564
 March 22, 2018: Verified to be fixed at least as of iOS 11.3b6.
 */
@property (nonatomic, readonly) NSString *pspdf_nullCharacterCleanedString;
```

In iOS 11, [Smart Punctuation](https://pspdfkit.com/blog/2018/ios-11-smart-punctuation/) did cause data loss when the dash key is tapped 3 times with the US English keyboard. We take data-loss issues very serious and quickly shipped a workaround for this OS-level bug.

# Radar Time

We have [strict rules for working around OS bugs](https://pspdfkit.com/blog/2016/writing-good-bug-reports/). One of them is requiring to submit a [b̶u̶g̶ ̶r̶e̶p̶o̶r̶t̶ Feedback Assistant entry](http://openradar.appspot.com/34651564) to Apple, to both report and document the issue. Since this issue was reported in 2017, it was later converted from radar to Feedback Assistant, into FB5735669.

![](/assets/img/2020/null-characters/feedback.png)

Apple did follow up on this issue to report that this has been reported before. However when I look at the header of Feedback Assistant, I see this:

```
Recent Similar Reports: None
Resolution: Open
```

Is it though? The comment in our codebase mentions that this is fixed. And in fact I am unable to reproduce this with and OS other than iOS 11.0. (We keep devices with older OS versions around so we can test)

It is unfortunate that Apple's bug reporting process is so dysfunctional. This ticket should show the correct state (closed) and that similar reports are attached, but something fell apart with either closing the original ticket or because of the mapping from radar to Feedback Assistant.

Another option would be to maintain a public database of known issues, or make it easier to share issues publicly. Apple does do that to a degree ([WebKit](https://bugs.webkit.org/show_bug.cgi?id=22102), [Swift](https://bugs.swift.org/browse/SR-6958), and in the Xcode release notes.) but not for OS-level issues.

We indeed can no longer reproduce this issue, and it seems to be fixed in iOS 11.1. But why are NULL bytes even a problem? They are invisible after all.

# NULL bytes in Strings? 

C-based strings are null-terminated. If we create a string in code, the compiler automatically appends `\0` at the end. 

```c
char cString[] = "A\0BC"; // 5 bytes in memory: A\0BC\0
printf("string: %s", cString); // outputs: string: A
```

`NSString` is more sophisticated; it is a [class cluster](https://developer.apple.com/library/archive/documentation/General/Conceptual/DevPedia-CocoaCore/ClassCluster.html) - using different representations[^1] depending on the content and toll-free bridged to `CFString`.

[^1]: [`__NSLocalizedString`](https://github.com/nst/iOS-Runtime-Headers/blob/master/Frameworks/Foundation.framework/__NSLocalizedString.h), [`NSLocalizableString`](https://github.com/nst/iOS-Runtime-Headers/blob/master/Frameworks/Foundation.framework/NSLocalizableString.h), [`NSPlaceholderString`](https://github.com/nst/iOS-Runtime-Headers/blob/master/Frameworks/Foundation.framework/NSPlaceholderString.h), `__NSCFConstantString`, `__NSCFString` and `NSTaggedPointerString` and more. [Tagged pointers](https://www.mikeash.com/pyblog/friday-qa-2015-07-31-tagged-pointer-strings.html) specifically are an interesting technology used to increase performance and reduce memory usage.

Depending on the backend, different backing stores are used. [`CFString` uses 16-bit unichars](https://developer.apple.com/library/archive/documentation/CoreFoundation/Conceptual/CFStrings/Articles/StringStorage.html#//apple_ref/doc/uid/20001179-CJBEJBHH) (UTF-16). `CFString` stores length in a variable, so accessing it is O(1) where C-strings need to iterate the string to find the null-termination `\0` character, making [`strlen`](https://overiq.com/c-programming-101/the-strlen-function-in-c/) an O(n) operation[^2].

[^2]: There are many interesting ways to speed up strlen. While Clang's libc uses [a straightforward loop with a todo for performance optimization](https://github.com/llvm/llvm-project/blob/master/libc/src/string/strlen.cpp), gcc's glibc [tests four bytes at a time](https://github.com/lattera/glibc/blob/master/string/strlen.c) ([explanation how this works](https://stackoverflow.com/a/57651888/83160)) and OpenBSD has [hand-optimized code for some architectures, such as arm64](https://github.com/openbsd/src/blob/master/lib/libc/arch/amd64/string/strlen.S).

### String in Swift

Swift has a modern [String object](https://developer.apple.com/documentation/swift/string) (e.g. using [grapheme clusters](https://makeapppie.com/2019/03/31/swift-strings-are-not-c-strings-or-nsstrings/) for better unicode/emoji support), but behaves very similar to Objective-C in the case of NULL.

```objc
NSString *string = @"A\0BC";
NSLog(string);                   // prints "ABC"
NSLog(@"%@", string);            // prints "A"
printf(string.UTF8String);       // prints "A"
```

Notice how Swift even gets the `NSLog` case right, whereas this cut off the string in the Objective-C example above.

```swift
let string = "A\u{0}BC";
print(string).                  // prints "ABC"
print("\(string)")              // prints "ABC"
NSLog("%@", string)             // prints "ABC"
NSLog("%@", string as NSString) // prints "ABC"

string.withCString { cstr in    // prints "A"
    print("\(String(cString: cstr)), len: \(strlen(cstr))")
}
```

However you generally want to avoid NULL characters in your strings. They trigger bugs: [SR-11355: String.range(of...) matches an extra NUL (and returns range outside string boundaries)](https://bugs.swift.org/browse/SR-11355). I'm sure we could find more issues with some digging.

### std::string in C++

The most common way to store strings in C++ is [`std::string`](http://www.cplusplus.com/reference/string/string/). Reading the implementation is [extremely hard](https://github.com/llvm-mirror/libcxx/blob/master/include/string), but there are [much more readable](https://joellaity.com/2020/01/31/string.html) explanation about its internals.

`std::string` operates on bytes not on characters or even graphemes. While there are different storage modes, both include a size field and do not rely on a null-termination. 

To bridge between Objective-C and C++, we use [djinni](https://github.com/dropbox/djinni), which uses Objective-C++ to conveniently translate between these types. Below code is slightly simplified, removing assertions:

```
static std::string toCpp(NSString* string) {
    return {string.UTF8String, [string lengthOfBytesUsingEncoding:NSUTF8StringEncoding]};
}

static NSString* fromCpp(const std::string& string) {
    return [[NSString alloc]
      initWithBytes:string.data()
      length:static_cast<NSUInteger>(string.size())
      encoding:NSUTF8StringEncoding];
}
```

This converts lossless between the two representations:

```cpp
NSString *string = @"A\0BC";
std::string cppString = toCpp(string);
std::cout << cppString;                  // prints ABC
NSString *nsString = fromCpp(cppString);
NSLog(nsString);                         // prints ABC
```

# Strings in PDF

Now why is `\0` still a problem? PDF has two ways to encode strings: Name and String. 

Name objects allow ASCII characters with the exception of `\0`, other characters are encoded, for example `#20` for space. This is generally used for keys and not longer-form text.

String objects are (usually) encoded between `(` and `)` and support all characters, including NULL bytes. However, we project both data types to `String/NSString` in our iOS SDK, to not burden developers with additional complexity. So depending on what property is accessed, `\0` would be silently discarded or not.

In practice we have some more parsing and optimized code that uses C strings for performance. Our parsing code runs everywhere, from [WebAssembly in modern browsers](https://pspdfkit.com/blog/2017/webassembly-a-new-hope/) to [Internet Explorer 11](https://pspdfkit.com/guides/web/current/customizing-the-interface/ie11/). In this layer, strings are truncated early when a `\0` is found.

# Conclusion

We decided to remove the code that would manually check for NULL characters at the API boundary - if developers really include NULL characters, we accept that the string is truncated. It's a bad idea to do, and this is a better tradeoff than reducing performance for everyone to work around an edge case, now that this can no longer be triggered from the User Interface.