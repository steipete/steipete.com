---
layout: post
title:  "null-character in strings, or why writing radars for Apple is such a frustrating experience"
date:   2020-05-24 10:00:00 +0200
tags: development iOS
---

We're finally dropping iOS 11 support the PSPDFKit SDK. To be honest, there's nothing major in iOS 12 that forces us to upgrade, after all the whole release was about "Bug Fixes & Performance Improvements".

# Motivation

There are however a variety of reasons to drop older versions of iOS:

- Test overhead
- Remove old hacks

# Our Process

PSPDFKit is an old codebase. There's code in there from 2008, and we're modernizing the codebase carefully to make sure it stays a comfortable place to work in.

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

Back in 2017 I stumbled on the issue via Twitter:

{% https://twitter.com/gabrielhauber/status/912590194099888128 %}

In retrospect it was extremely useful to find this tweet, since we go reports about sporadic data-loss on note annotations, which ended up being this very issue.

# Radar Time

We have [strict rules for working around OS bugs](https://pspdfkit.com/blog/2016/writing-good-bug-reports/). One of them is requiring to submit a [b̶u̶g̶ ̶r̶e̶p̶o̶r̶t̶ Feedback Assistant entry](http://openradar.appspot.com/34651564) to Apple. This is great because now I can read up on it to verify the status. This was reported before there was Feedback Assistant, Apple converted this into FB5735669. 

![](/assets/img/2020/null-characters/feedback.png)

Apple did follow up to report that this has been reported before. However when I look at the header of Feedback Assistant, I see this:

```
Recent Similar Reports: None
Resolution: Open
```

Is it though? The comment in our codebase says different. And in fact I am unable to reproduce this with and OS other than iOS 11.0. (We keep devices with older OS versions around so we can test)

# NULL characters in NString?

C-based strings are null-terminated. If we create a string in code, the compiler automatically appends `\0` at the end. 

```c
char cString[] = "A\0BC"; // 5 bytes in memory: A\0BC\0
printf("string: %s", cString); // outputs: string: A
```

`NSString` is more sophisticated; it is a [class cluster](https://developer.apple.com/library/archive/documentation/General/Conceptual/DevPedia-CocoaCore/ClassCluster.html) - using different representations depending on the content. It is toll-free bridged to CFString, and the most common types[^2] are `__NSCFConstantString`, `__NSCFString` and `NSTaggedPointerString`[^1].

[^1]: While not relevant for this article, [Tagged pointers](https://www.mikeash.com/pyblog/friday-qa-2015-07-31-tagged-pointer-strings.html) are an interesting technology used to increase performance and reduce memory usage.

[^2]: [`__NSLocalizedString`](https://github.com/nst/iOS-Runtime-Headers/blob/master/Frameworks/Foundation.framework/__NSLocalizedString.h), [`NSLocalizableString`](https://github.com/nst/iOS-Runtime-Headers/blob/master/Frameworks/Foundation.framework/NSLocalizableString.h), [`NSPlaceholderString`](https://github.com/nst/iOS-Runtime-Headers/blob/master/Frameworks/Foundation.framework/NSPlaceholderString.h)

```
(lldb) po [[[NSString alloc] initWithString:@"A\0BC"] class] // NSTaggedPointerString
(lldb) po [[[NSString alloc] initWithString:@"ABCDEFGHIJK"] class] // __NSCFString
(lldb) po [[[NSString alloc] initWithString:@"🦥"] class] // __NSCFString
```

Depending on the backend, different backing stores are used. [CFString uses 16-bit unichars](https://developer.apple.com/library/archive/documentation/CoreFoundation/Conceptual/CFStrings/Articles/StringStorage.html#//apple_ref/doc/uid/20001179-CJBEJBHH) (UTF-16).

CFString stores length in a variable, so accessing it is O(1) where C-strings need to iterate the string to find the null-termination `\0` character, making [`strlen`](https://overiq.com/c-programming-101/the-strlen-function-in-c/) an O(n) operation[^3].

[^3]: There are many interesting ways to speed up strlen. While Clang's libc uses [a straightforward loop with a todo for performance optimization](https://github.com/llvm/llvm-project/blob/master/libc/src/string/strlen.cpp), gcc's glibc [tests four bytes at a time](https://github.com/lattera/glibc/blob/master/string/strlen.c) ([explanation how this works](https://stackoverflow.com/a/57651888/83160)) and OpenBSD has [hand-optimized code for some architectures, such as arm64](https://github.com/openbsd/src/blob/master/lib/libc/arch/amd64/string/strlen.S).

### String in Swift
This is important when we try to understand the following, somewhat surprising behavior. String prints differently depending on how we output it:

```objc
NSString *string = @"A\0BC";
NSLog(string);                   // prints "ABC"
NSLog(@"%@", string);            // prints "A"
printf(string.UTF8String);       // prints "A"
printf("%s", string.UTF8String); // prints "A"
```

Let's look at Swift next. The null character is preserved, until we do a roundtrip via `.withCString()`

```swift
let string = "A\0BC";
print(string).                  // prints "ABC"
print("\(string)")              // prints "ABC"
NSLog("%@", string)             // prints "ABC"
NSLog("%@", string as NSString) // prints "ABC"

string.withCString { cstr in    // prints "A"
    print("\(String(cString: cstr)), len: \(strlen(cstr))")
}
```

Swift has a more modern handling of Strings (e.g. it uses [grapheme clusters](https://makeapppie.com/2019/03/31/swift-strings-are-not-c-strings-or-nsstrings/) for better unicode/emoji support), but there are no difference when converting to C.

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

Let's do a roundtrip of our string:

```cpp
NSString *string = @"A\0BC";
std::string cppString = toCpp(string);
std::cout << cppString;                  // prints ABC
NSString *nsString = fromCpp(cppString);
NSLog(nsString);                         // prints ABC
```

Great - we convert lossless between the two representations. Now why is `\0` still a problem?

# Strings in PDF

PDF has two ways to encode strings: Name and String.






# What Could Apple Improve?

- Fix the bug reporting process. It seems someone actually did read the ticket, linked it to another one, but then things did fell apart with either closing the original ticket or because of the mapping from radar to Feedback Assistant.

- Maintain a public database of known issues, or make it easier to share issues publicly. Apple does do that to a degree ([WebKit](https://bugs.webkit.org/show_bug.cgi?id=22102), [Swift](https://bugs.swift.org/browse/SR-6958), and in the Xcode release notes.)

# Conclusion

Because we are careful to report bugs 