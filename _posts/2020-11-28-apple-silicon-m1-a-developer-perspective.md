---
layout: post
title: "Apple Silicon M1: A Developer's Perspective"
date: 2020-11-28 15:00:00 +0200
tags: iOS development
image: /assets/img/2020/m1/m1.jpg
description: "The excitement around Apple's new M1 chip is everywhere. I bought a MacBook Air 16GB M1 to see how viable it is as main development machine."
---

The excitement around Apple's new M1 chip is [everywhere](https://www.singhkays.com/blog/apple-silicon-m1-black-magic/). I bought a MacBook Air 16GB M1 to see how viable it is as main development machine - here's an early report after a week of testing.

## Xcode

Xcode runs FAST on the M1. Compiling the [PSPDFKit PDF SDK](https://pspdfkit.com/) (debug, arm64) can almost compete with the fastest Intel-based MacBook Pro Apple offers to date, with [8:49 min vs 7:31 min](https://twitter.com/steipete/status/1332052251712614405?s=21). For comparison, my Hackintosh builds the same in less than 5 minutes. 

One can't overstate how impressive this is for a fan-less machine. Apple's last experiment with fan-less MacBooks was the 12-inch version from 2017, which builds the same project in 41 minutes.

Our tests mostly ran just fine, although I found [a bug specific to arm64](https://github.com/Aloshi/dukglue/pull/27) that we missed before, as we don't run our tests on actual hardware on CI. Moving the Simulator to the same architecture as shipping devices will be beneficial and will help find more bugs.

Testing iOS below 14 is problematic. It seems [WebKit is crashing in a memory allocator](https://twitter.com/steipete/status/1332654247809257473?s=21), throwing EXC_BAD_INSTRUCTION (code=EXC_I386_INVOP, subcode=0x0) (Apple folks: FB8920323). Performance also seems really bad, with Xcode periodically [freezing](https://twitter.com/steipete/status/1332348616145563653?s=21) and the whole system becoming so [slow](https://twitter.com/steipete/status/1332648748158246922?s=21) that the mouse cursor gets choppy. Some Simulators even make problems on iOS 14, [such as the iPad Air (4th gen) which still emulates Intel](https://twitter.com/steipete/status/1331628274783543297?s=21), so try to avoid that one.

We were extremely excited to be moving our CI to Mac Mini's with M1 chip and are [waiting on MacStadium to release devices](https://www.macstadium.com/m1-mini), however it seems we will have to restrict tests to iOS 14 for that to work. With our current schedule, we plan to drop iOS 12 in Q3 2021 and iOS 13 in Q3 2022, so it will be a while until we can fully move to Apple Silicon.

There is a chance that Apple fixes these issues, however it's not something to count on - given that this only affects older versions of iOS, the problem will at some point just "go away".

**Update:** We're working around the WebKit crashes for now via detecting Rosetta2 translation at runtime and simply skipping the tests where WebKit is used. This isn't great, but luckily we're not using WebKit a lot in our current project. [See my Gist for details](https://gist.github.com/steipete/e15b1fabffc7da7d49c92e3fbd06971a). Performance seems acceptable if you restrict parallel testing to at most two instances - else the system simply runs out of RAM and swapping is just really slow.

**Update 2**: I've heard that the choppy mouse cursor is an Xcode/Simulator bug, and is currently being worked on. Workaround: Ensure at least one Simulator window is on-screen and visible.

## Docker

We use Docker to automate our Website and load environments for our [Web and Server PDF SDKs](https://pspdfkit.com/pdf-sdk/web/). Docker posted a [status update blog post](https://www.docker.com/blog/apple-silicon-m1-chips-and-docker/) about the current state of things, admitting that it currently won't work but that they're [working on it](https://github.com/docker/roadmap/issues/142). There are more [hacky ways to use Apple's Hypervisor to run Docker container manually](https://finestructure.co/blog/2020/11/27/running-docker-on-apple-silicon-m1-follow-up), however this needs arm-based containers.

I expect a solution in Q1 2021 that runs arm-based containers. We'll have to do some work to add arm-support (something already on the roadmap) so this is only a transitional issue.

## Virtualization and Windows

To test our [Windows PDF SDK](https://pspdfkit.com/pdf-sdk/windows/), most folks are using a VMware virtual machine with Windows 10 and Visual Studio. Currently none of the Mac virtualisation solutions support Apple Silicon, however both [VMware and Parallels](https://appleinsider.com/articles/20/11/11/parallels-confirms-apple-m1-support-amid-silence-from-other-virtualization-companies) are working on it. I do not expect Virtualbox to be updated [anytime soon](https://forums.virtualbox.org/viewtopic.php?f=8&t=98742).

I expect that eventually we'll be able to run ARM-based Windows with commercial tooling. Various [proof-of-concepts](https://9to5mac.com/2020/11/27/arm-windows-virtualization-m1-mac/) already exist, and performance seems [extremely promising](https://twitter.com/imbushuo/status/1332772957609922561?s=21). Microsoft currently doesn't sell ARM-based Windows, so getting a license will be interesting.

ARM-Windows can emulate x86 applications, and Microsoft is working on [x64 emulation](https://www.neowin.net/news/it039s-official-x64-emulation-is-coming-to-windows-on-arm), which is already rolling out in Insider builds. In a few months, it should be possible to develop and test our Windows SDK with Visual Studio on M1 in reasonable performance.

Running older versions of macOS might be more problematic. We currently support macOS 10.14 with our [AppKit PDF SDK](https://pspdfkit.com/blog/2017/pspdfkit-for-macos/) and macOS 10.15 with  the [Catalyst PDF SDK](https://pspdfkit.com/blog/2019/pspdfkit-for-mac-catalyst/), both OS releases that require testing. It remains to be seen if VMware or Parallels include a complete x64 emulation layer. This would likely be really slow, so I wouldn't  count on it.

![](/assets/img/2020/m1/memory.png)

Lastly, 16 GB RAM just isn't a lot. When running parallel tests, the machine starts to heavily swap and performance really goes down the drain. This will be even more problematic with virtual machines running. Future machines will offer 32 GB options to alleviate this issue.

**Update:** [How to run Windows 10 on ARM in Qemu with Hypervisor.framework patches on Apple Silicon Mac](https://gist.github.com/niw/e4313b9c14e968764a52375da41b4278#file-readme-md)

## Android Studio

IntelliJ is working on porting the [JetBrains Runtime](https://youtrack.jetbrains.com/issue/JBR-2526) to Apple Silicon. The apps currently work through Rosetta 2, however building via Gradle is [extremely slow](https://www.reddit.com/r/androiddev/comments/jx4ntt/apple_macbook_air_m1_is_very_slow_in_gradle_builds/). Gradle creates code at runtime, which seems a particular bad combination with the Rosetta 2 ahead-of-time translation logic. 

I expect that most issues will be solved by Q1 2021, however it will likely be some more time until all Java versions run great on ARM. A lot of effort has been put into [loop unrolling and vectorisation](https://bell-sw.com/java/arm/performance/2019/01/15/the-status-of-java-on-arm/), not everything there is available on ARM just yet.

**Update:** [Azul offers macOS JDKs for arm64](https://www.azul.com/press_release/azul-announces-support-of-java-builds-of-openjdk-for-apple-silicon/), including for [Java 8](https://www.azul.com/downloads/zulu-community/?os=macos&architecture=arm-64-bit&package=jdk).

## Homebrew

[Homebrew](https://brew.sh/) currently works via Rosetta 2. Just prefix everything with `arch -x86_64` and it'll just work. It is possible to install an additional (arm-based) version of Homebrew [under `/opt/homebrew`](https://soffes.blog/homebrew-on-apple-silicon) and mix setup, as [more and more software](https://github.com/Homebrew/brew/issues/7857) is adding support for arm.

This is not a problem currently (performance is good) and will eventually just work natively.

## Applications

Most applications just work, Rosetta is barely noticeable. Larger apps to take a longer initial performance hit (e.g. Microsoft Word takes [around 20 seconds](https://www.zdnet.com/article/microsoft-office-will-be-about-20-second-slower-initially-on-apple-silicon-rosetta-2/) until everything is translated), but then these binaries are cached and subsequent runs are fast.

There's the occasional app that can't be translated and fails on startup (e.g. [Beamer](https://beamer-app.com/download) or the [Google Drive "Backup and Sync" client](https://www.google.com/intl/en_gh/drive/download/)), but this is rare. Some apps are confused about their place on disk and ask to be moved to the Applications directory, when really it's just the translated binary that runs somewhere else. Most of these dialogs can be ignored. Some apps (e.g. Visual Studio Code) [block auto-updating](https://twitter.com/steipete/status/1331884524934995968?s=21) as the translated app location is readonly. However, in case of VS Code, the Insider build is already updated to ARM and just works.

Electron-based apps are slow if they run on Rosetta. It seems the highly optimized V8 JavaScript compiler blocks ahead-of-time translation. The latest stable version of Electron (Version 11) already [fully supports Apple Silicon](https://www.electronjs.org/blog/apple-silicon), and companies like Slack already updated their beta version to run natively.

Google just shipped [Chrome that runs on ARM](https://www.macworld.com/article/3597749/google-releases-chrome-87-with-support-for-apple-silicon-macs.html), however there's still quite a performance gap between it and Apple Safari, which just *flies* on Apple Silicon.

## Conclusion

The new M1 MacBooks are fast, beautiful and silent and the hype is absolutely justified. There's still a lot to do on the software-front to catch up, and the bugs around older iOS Simulators are especially problematic.

All of that can be fixed in software and the whole industry is currently working on making the experience better, so by next year, when Apple updates the 16-inch MacBook Pro and releases the next generation of their M chip line, it should be absolutely possible to use a M1 Mac as main dev machine.

For the time being, the M1 will be my <del>travel</del> secondary laptop, and I'll keep working on the 2,4 GHz 16-inch MacBook Pro with 32 GB RAM, which just is the faster machine. I'll be much harder to accept the loud, always-on fans though, now that I know what soon will be possible.