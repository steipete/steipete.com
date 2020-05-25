---
layout: post
title:  "Jailbreaking for iOS Developers"
date:   2020-05-21 10:30:00 +0200
tags: iOS hacks
image: https://twitter.com/mnzthegreat/status/1264848209735585792/photo/1
---

Jailbreaking is something that's rarely discussed in the iOS developer community. Which is unfortunate - because it's amazing. Let's walk through a few useful things you can do with it.

# Definition & Legality

>iOS jailbreaking is a privilege escalation to remove software restrictions imposed by Apple on iOS, tvOS, and watchOS. This is done through a series of kernel patches. This jailbreaking allows root access to iOS, allowing the downloading and installing of additional applications and extensions which are unavailable through the official Apple App Store. ([Source](https://www.techacrobat.com/ios-12-4-unc0ver-jailbreak/))

If you're wondering, [is this legal](https://en.wikipedia.org/wiki/IOS_jailbreaking#Legality)? This depends on your country, but it is legal in Austria, Germany, Canada, India, New Zealand, United Kingdom and the United States.

With the exception of the latest iOS 13.5 jailbreak, all major jailbreaks since 10 were based on publicly available exploits, which were [reported to Apple before](https://twitter.com/helthydriver/status/1265024132204331011?s=21).

# Motivation

Jailbreaking has a bad taste because it can be used to pirate apps. But there are many other, much more noble or interesting reasons why it's worth exploring, like enhancing accessibility:

{% twitter https://twitter.com/gadgetgal_/status/1264962311229333504 %}

- Security Research (MITM via [SSLKillSwitch](https://github.com/nabla-c0d3/ssl-kill-switch2), [dumpdecrypted](https://github.com/stefanesser/dumpdecrypted) to decrypt/disassemble apps)
- Enhancing iOS with tweaks (e.g. [native picture-in-picture](https://repo.packix.com/package/codes.rambo.ipadify/), [tabs in Safari](https://repo.twickd.com/package/com.twickd.minazuki.safari-electro-2))
- Enhancing other apps (e.g. adding a [Tomatometer score to Netflix](https://repo.packix.com/package/org.packix.flixenhancer/))
- Installing apps that are not allowed on the App Store because they could be used for copyright violations ([Game Console Emulators](https://tweak-box.com/delta/) or are only legal in certain countries ([Automatic Call Recording](http://ioscallrecorder.com/))
- Theming via [SnowBoard](https://repo.packix.com/package/com.spark.snowboard/) ([Viola](https://repo.packix.com/package/com.bousrih.viola/) is what you see on the screenshot above)
- Convenience for developers (ssh into the device, inspect view hierarchy)
- Accessing Hardware features that are otherwise inaccessible (e.g. [NFC Writer](http://cydia.saurik.com/package/net.limneos.nfcwriterx/))
- Improving accessibility (e.g. [blind folks], [special needs](https://twitter.com/GadgetGal_/status/1264962311229333504?s=20))

Beware however that some apps (think Banking) might include a jailbreak detection and won't work if they detect Cydia.

# State of Jailbreaking

It has never been a better time for jailbreaking. From iOS 10-13, almost every version can be hacked, including the just-released iOS 11.5.

Reddit maintains [a great overview wiki]((https://www.reddit.com/r/jailbreak/wiki/escapeplan/guides/jailbreakcharts#wiki_ios13.x)) on the current jailbreak availability situation. The two interesting ones (as of May 2020) are:

- [checkra1n](https://checkra.in/) uses the [Checkm8 exploit](https://arstechnica.com/information-technology/2019/09/developer-of-checkm8-explains-why-idevice-jailbreak-exploit-is-a-game-changer/) - an unpatchable vulnerability in the  iOS bootrom for all devices from A5-A11 (everything up to iPhone X)

- [unc0ver](https://unc0ver.dev/), a semi-untethered jealbreak using various hacks, just updated for iOS 13.5.

Both are [semi-tethered jailbreaks](https://www.idownloadblog.com/2019/11/21/types-of-jailbreaks/). You need to re-trigger the jailbreak after a reboot to patch the kernel, so it can run unsigned code.

# Cydia 101

Cydia is the oldest and most common alternative App Store for iOS. It used to also sell apps, but now focusses on simply being a UI for the apt-get package manager. Apps are found via repositories, the two most popular ones are however not preinstalled:

- [Packix](https://repo.packix.com/)
- [Dynastic Repo](https://repo.dynastic.co/)

Both offer free and paid apps (via PayPal or Credit Card) and can be easily added to Cydia.

Heads up: Cydia hosts many tweaks that are outdated and won't work on iOS 13 anymore. Better check [/r/jailbreak](https://www.reddit.com/r/jailbreak/) or [iDownloadBlog](https://www.idownloadblog.com/tag/jailbreak/) to find tweaks that work.

# Preserve SHSH2 Blobs

A [SHSH blob](https://en.wikipedia.org/wiki/SHSH_blob) is a small piece of data that is part of Apple's digital signature protocol for iOS restores and updates.

As of writing this post, Apple signs iOS 13.4.1 and iOS 13.5, and you can expect that they remove 13.4.1 in a few days. With saving this blob, you can downgrade at any time, without being dependent on Apple. 

![](/assets/img/2020/jailbreaking/blobsaver.png)

There are many ways how you can save these. I recommend [blobsaver](https://github.com/airsquared/blobsaver/releases), as it saves the blobs on disk, instead of relying on cloud services. Tools like [futurerestore](https://github.com/tihmstar/futurerestore) can then be used to downgrade ([read more here](https://cellularnews.com/mobile-operating-systems/how-to-downgrade-ios-using-shsh2-blobs/)). Store them - you never know when they might come in handy.

With that out of the way, let's explore what we can all do with our new superpowers:

# SSL Kill Switch

[SSLKillSwitch 2](https://github.com/nabla-c0d3/ssl-kill-switch2) is a tweak to disable SSL certificate validation on device. This is useful to see what data apps send via a MITM proxy. 

- Install PreferenceLoader (dependency) and the [Filza](https://filza.net/) (File Browser) on Cydia
- Download the [latest version from GitHub](https://github.com/nabla-c0d3/ssl-kill-switch2/releases) (deb file).
- Open Filza, navigate to `/private/var/mobile/Library/Mobile Documents/com~apple~CloudDocs/Downloads`
- Open the downloaded `com.nablac0d3.sslkillswitch2_0.14.deb` (or similar) and press Install (top right button)
- Respring (Restart SpringBoard)
- Find SSLKillSwitch 2 in iOS Settings

If you're curious how this works on a technical level, here's a [writeup for iOS 12](https://nabla-c0d3.github.io/blog/2019/05/18/ssl-kill-switch-for-ios12/), or just explore the [source on GitHub]((https://github.com/nabla-c0d3/ssl-kill-switch2).

# FLEX In-App Debugging

[FLEX](https://github.com/Flipboard/FLEX) is an open-source in-app debugging and exploration tool for iOS. It's amazing what you can do with it. Want the weather background as home screen background? No problem.

{% twitter https://twitter.com/nsexceptional/status/1250353513923674114 %}

To Install, download [FLEXing](http://cydia.saurik.com/package/com.pantsthief.flexing/), reboot and then tap on the status bar to load FLEX. You can browse the classes and inspect the view hierarchy with a 3D debugger, similar to Reveal:

![](/assets/img/2020/jailbreaking/hierarchy-spotify.png)

Of course you can also inspect apps written in SwiftUI, like the popular [ACHNBrowserUI](https://github.com/Dimillian/ACHNBrowserUI).

![](/assets/img/2020/jailbreaking/hierarchy-achelper.png)

# More Useful Cydia Apps & Tweaks

- [iPadify](https://repo.packix.com/package/codes.rambo.ipadify/) - install iPad-only app such as Playgrounds, native picture-in-picture
- [Prysm](https://repo.packix.com/package/com.laughingquoll.prysm/) - a feature-rich control center for iOS
- [Revealloader](https://github.com/heardrwt/RevealLoader): Load reveal to any app



- Frida: Code injection
- Supercharge: Creating simple tweaks on the device
• TetherMe — so you don't have to pay for a tethering plan
• Barmoji — add an emoji bar, pictured below
• DockX — add quick actions below the keyboard, also pictured
• Snapper 2 — crop screenshots before taking them
• SwipeSelection — swipe on the keyboard to move the cursor
• FrontCamUnmirror — self explanatory
• TapTapFlip — double tap to flip the camera
• SnapMoreText — press enter to add a new line to Snapchat text boxes
• Various tweaks to remove ads in YouTube and other apps
- copilot https://repo.packix.com/package/me.tomt000.copylog/

https://twitter.com/avimanyuroy3/status/1264346815165431809?s=21
https://twitter.com/avimanyuroy3/status/1264346815165431809?s=21

https://www.idownloadblog.com/2019/04/04/icaughtu-12/


Velox Reloaded, HomePlus Beta or HomePlus Pro, Prysm, Notifica

https://twitter.com/snazzyq/status/1264767792663822336?s=21

https://twitter.com/nsexceptional/status/1264394490703462407?s=21

Copilot
Shortcuts: True cuts, Activator 


# Themes


https://twitter.com/snazzyq/status/1264767736812457986?s=21

# Examples

https://twitter.com/ThisIsNoahEvans/status/1264541020055879682?s=20

https://twitter.com/iM4CH3T3/status/1264797535316709377?s=20
https://twitter.com/V3RSAC3THUG/status/1264688114569805825?s=20

Mignon theme?

RevaUI
https://twitter.com/alolastmoment/status/1264340002592509953?s=20

https://twitter.com/alolastmoment/status/1264340002592509953?s=20

https://www.xda-developers.com/apple-iphone-ipad-ios-ipados-jailbreak-unc0ver/

https://www.wired.com/story/apple-ios-unc0ver-jailbreak/

# Cydia Repos

- [Packix](https://repo.packix.com/)
- [Dynastic Repo](https://repo.dynastic.co/) (tweaks)