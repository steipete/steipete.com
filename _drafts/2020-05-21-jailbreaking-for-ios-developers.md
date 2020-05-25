---
layout: post
title:  "Jailbreaking for iOS Developers"
date:   2020-05-21 10:30:00 +0200
tags: iOS hacks
---

Jailbreaking is something that's rarely discussed in the iOS developer community. Which is unfortunate - because it's amazing. Let's walk through a few useful things you can do with it.

# Definition & Legality

>iOS jailbreaking is a privilege escalation to remove software restrictions imposed by Apple on iOS, tvOS, and watchOS. This is done through a series of kernel patches. This jailbreaking allows root access to iOS, allowing the downloading and installing of additional applications and extensions which are unavailable through the official Apple App Store. ([Source](https://www.techacrobat.com/ios-12-4-unc0ver-jailbreak/))

If you're wondering, [is this legal](https://en.wikipedia.org/wiki/IOS_jailbreaking#Legality)? This depends on your country, but it is legal in Austria, Germany, Canada, India, New Zealand, United Kingdom and the United States.

# Motivation

Jailbreaking has a bad taste because it can be used to pirate apps. But there are many other, much more noble or interesting reasons why it's worth exploring, like enhancing accessibility:

{% twitter https://twitter.com/gadgetgal_/status/1264962311229333504 %}

- Security Research (MITM via SSLKillSwitch, dumpdecrypted to disassemble apps)
- Enhancing iOS with tweaks (e.g. native picture-in-picture in Safari, TV and other places)
- Enhancing other apps (e.g. adding a [Tomatometer score to Netflix](https://repo.packix.com/package/org.packix.flixenhancer/))
- Installing apps that are not allowed on the App Store because they could be used for copyright violations ([Game Console Emulators](https://tweak-box.com/delta/) or are only legal in certain countries ([Automatic Call Recording](http://ioscallrecorder.com/))
- Convenience for developers (ssh into the device, inspect view hierarchy)
- Accessing Hardware features that are otherwise inaccessible (e.g. [NFC Writer](http://cydia.saurik.com/package/net.limneos.nfcwriterx/))
- Improving accessibility (e.g. [blind folks], [special needs](https://twitter.com/GadgetGal_/status/1264962311229333504?s=20))

Beware however that some apps (think Banking) might include a jailbreak detection and won't work if they detect Cydia.

# State of Jailbreaking

It has never been a better time for jailbreaking. From iOS 10-13, almost every version can be hacked, including the just-released iOS 11.5.

Reddit maintains [a great overview wiki]((https://www.reddit.com/r/jailbreak/wiki/escapeplan/guides/jailbreakcharts#wiki_ios13.x)) on the current jailbreak availability situation. The two interesting ones (as of May 2020) are:

- [checkra1n](https://checkra.in/) uses the [Checkm8 exploit](https://arstechnica.com/information-technology/2019/09/developer-of-checkm8-explains-why-idevice-jailbreak-exploit-is-a-game-changer/) - an unpatchable vulnerability in the  iOS bootrom for all devices from A5-A11 (everything up to iPhone X)

- [unc0ver](https://unc0ver.dev/), a semi-untethered jealbreak using various hacks, just updated for iOS 13.5.

# Cydia 101

Cydia is the oldest and most common alternative App Store for iOS. It used to also sell apps, but now focusses on simply being a UI for the apt-get package manager. Apps are found via repositories, the two most popular ones are however not preinstalled:

- [Packix](https://repo.packix.com/)
- [Dynastic Repo](https://repo.dynastic.co/)

Both offer free and paid apps (via PayPal or Credit Card) and can be easily added to Cydia.

# Preserve SHSH2 Blobs

TODO


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

It also comes with a 3D Inspector, similar to Reveal:

{% twitter https://twitter.com/nsexceptional/status/1218194136446177282 %}

To Install, get ...


- Frida: Code injection
- Revealloader: Load reveal to any app
- Supercharge: Creating simple tweaks on the device

# Useful Cydia Apps & Tweaks

- [iPadify](https://repo.packix.com/package/codes.rambo.ipadify/) - install iPad-only app such as Playgrounds, native picture-in-picture
- [Prysm](https://repo.packix.com/package/com.laughingquoll.prysm/) - a feature-rich control center for iOS




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