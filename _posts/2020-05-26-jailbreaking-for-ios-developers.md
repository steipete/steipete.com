---
layout: post
title:  "Jailbreaking for iOS Developers"
date:   2020-05-26 00:00:00 +0200
tags: iOS hacks
image: /assets/img/2020/jailbreaking/header.jpg
---

<style type="text/css">
div.post-content > img:first-child { width:50% !important; }
</style>

Jailbreaking is something that's rarely discussed in the iOS developer community. Which is unfortunate - because it's amazing. Let's walk through a few useful things you can do with it. (Picture by [@mnzthegreat](https://twitter.com/mnzthegreat/status/1264848209735585792))

# Definition & Legality

>iOS jailbreaking is a privilege escalation to remove software restrictions imposed by Apple on iOS, tvOS, and watchOS. This is done through a series of kernel patches. This jailbreaking allows root access to iOS, allowing the downloading and installing of additional applications and extensions which are unavailable through the official Apple App Store. ([Source](https://www.techacrobat.com/ios-12-4-unc0ver-jailbreak/))

If you're wondering, [is this legal](https://en.wikipedia.org/wiki/IOS_jailbreaking#Legality)? This depends on your country, but it is legal in Austria, Germany, Canada, India, New Zealand, United Kingdom and the United States. With the exception of the latest iOS 13.5 jailbreak, all major jailbreaks since 10 were based on publicly available exploits, which were [reported to Apple before](https://twitter.com/helthydriver/status/1265030817618767875?s=21).

If you don't want to risk jailbreaking, you can still install some apps with [AltStore](https://altstore.io/), which signs apps on your Mac.

# Motivation

Jailbreaking has a bad taste because it can be used to pirate apps. But there are many other, much more noble or interesting reasons why it's worth exploring, like enhancing accessibility:

<blockquote class="twitter-tweet" data-conversation="none"><p lang="en" dir="ltr">I have a special needs son for whom I jailbreak iDevices to support his disability. It is life changing for us.</p>&mdash; GadgetGal (@GadgetGal_) <a href="https://twitter.com/GadgetGal_/status/1264952195402723328?ref_src=twsrc%5Etfw">May 25, 2020</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

- Security Research (MITM via [SSLKillSwitch](https://github.com/nabla-c0d3/ssl-kill-switch2), [dumpdecrypted](https://github.com/stefanesser/dumpdecrypted) to decrypt/disassemble apps)
- Enhancing iOS with tweaks (e.g. [native picture-in-picture](https://repo.packix.com/package/codes.rambo.ipadify/), [tabs in Safari](https://repo.twickd.com/package/com.twickd.minazuki.safari-electro-2), a [call-bar](https://www.idownloadblog.com/2019/03/06/callbar-xs-brings-everyones-favorite-phone-call-centric-jailbreak-tweak-to-ios-12/), [better notifications](https://www.idownloadblog.com/2019/02/25/notifica/), [better shortcuts](http://cydia.saurik.com/package/com.ethanrdoesmc.truecuts/))
- Enhancing other apps (e.g. adding a [Tomatometer score to Netflix](https://repo.packix.com/package/org.packix.flixenhancer/) or disabling ads for YouTube [not linking, as this circumvents something that Google charges money for])
- Installing apps that are not allowed on the App Store because they could be used for copyright violations ([Game Console Emulators](https://tweak-box.com/delta/) or are only legal in certain countries ([Automatic Call Recording](http://ioscallrecorder.com/))
- Widgets on the Lock/Home Screen. (e.g. [Xen HTML](https://xenpublic.incendo.ws/))
- Theming via [SnowBoard](https://repo.packix.com/package/com.spark.snowboard/) ([Viola](https://repo.packix.com/package/com.bousrih.viola/) is what you see on the screenshot above, or [Mojito](https://repo.packix.com/package/eu.bednarz.eyeris/))
- Convenience for developers ([ssh into the device](https://twitter.com/develobile/status/1264656302195818497?s=21), [inspect view hierarchy](https://github.com/Flipboard/FLEX), a [proper file browser](http://cydia.saurik.com/package/com.tigisoftware.filza/))
- Accessing Hardware features that are otherwise inaccessible (e.g. [NFC Writer](http://cydia.saurik.com/package/net.limneos.nfcwriterx/))
- [Location spoofing](https://www.reddit.com/r/jailbreak/comments/dzdzgg/tutorial_nepetas_relocate_guide_on_1322/)
- Working around tethering restrictions ([TetherMe](http://cydia.saurik.com/package/net.tetherme.tetherme8/))
- Improving accessibility (e.g. [blind folks](https://twitter.com/devinprater/status/1264962317609046017), [special needs](https://twitter.com/GadgetGal_/status/1264962311229333504?s=20))

Many tweaks either modify apps itself or integrate into Settings:

<img src="/assets/img/2020/jailbreaking/settings.jpg" width="80%">

Beware: some apps (like banking) might include a jailbreak detection and won't work if they detect Cydia. However this also can be circumvented with the right tweak.

# State of Jailbreaking

There has [never been a better time](https://www.wired.com/story/apple-ios-unc0ver-jailbreak/) for jailbreaking. From iOS 10-13, almost every version can be hacked, including the just-released iOS 13.5. This is also somewhat worrying, as exploits require security flaws, and we're now at a stage where exploit platforms [aren't paying for any further exploits](https://9to5mac.com/2020/05/14/zerodium-has-too-many-ios-exploits/), because they already have [too many](https://twitter.com/cBekrar/status/1260543284008456192).

Reddit maintains [a great overview](https://www.reddit.com/r/jailbreak/wiki/escapeplan/guides/jailbreakcharts#wiki_ios13.x) on the current jailbreak availability situation. The two interesting ones (as of May 2020) are:

- [checkra1n](https://checkra.in/) uses the [Checkm8 exploit](https://arstechnica.com/information-technology/2019/09/developer-of-checkm8-explains-why-idevice-jailbreak-exploit-is-a-game-changer/) - an unpatchable vulnerability in the  iOS bootrom for all devices from A5-A11 (everything up to iPhone X)

- [unc0ver](https://unc0ver.dev/), a semi-untethered jailbreak using various hacks, just updated for iOS 13.5.

Both are [semi-tethered jailbreaks](https://www.idownloadblog.com/2019/11/21/types-of-jailbreaks/). You need to re-trigger the jailbreak after a reboot to patch the kernel, so it can run unsigned code. I recommend using [AltStore](https://altstore.io/) to install the Jailbreak. ([guide](https://www.idownloadblog.com/2020/02/16/how-to-unc0ver-altstore/))

# Adding Repositories to Cydia

[Cydia](https://cydia-app.com/) is the oldest and most common alternative App Store for iOS. It's automatically installed for most jailbreaks. It is a convenient UI for the apt-get packager that it comes with it. In the earlier days you could also buy apps through Cydia, nowadays most apps are sold via 3rd-party repositories. They offer free and paid apps (via PayPal or Credit Card) and can be easily added to Cydia.

- [Packix](https://repo.packix.com/)
- [Dynastic Repo](https://repo.dynastic.co/)
- [Twickd](https://repo.twickd.com/)

Heads up: Cydia hosts many tweaks that are outdated and won't work on iOS 13 anymore. Better check [/r/jailbreak](https://www.reddit.com/r/jailbreak/) or [iDownloadBlog](https://www.idownloadblog.com/tag/jailbreak/) to find tweaks that work.

# Preserve SHSH2 Blobs

A [SHSH blob](https://en.wikipedia.org/wiki/SHSH_blob) is a small piece of data that is part of Apple's digital signature protocol for iOS restores and updates.

As of writing this post, Apple signs iOS 13.4.1 and iOS 13.5, and you can expect that they remove 13.4.1 in a few days. With saving this blob, you can downgrade at any time, without being dependent on Apple. 

![](/assets/img/2020/jailbreaking/blobsaver.png)

There are many ways how you can save these. I recommend [blobsaver](https://github.com/airsquared/blobsaver/releases), as it saves the blobs on disk, instead of relying on cloud services. Tools like [futurerestore](https://github.com/tihmstar/futurerestore) can then be used to downgrade ([read more here](https://cellularnews.com/mobile-operating-systems/how-to-downgrade-ios-using-shsh2-blobs/)). Store them - you never know when they might come in handy.

With that out of the way, let's explore what we can all do with our new superpowers:

# SSL Kill Switch

[SSLKillSwitch 2](https://github.com/nabla-c0d3/ssl-kill-switch2) is a tweak to disable SSL certificate validation on device. This is useful to see what data apps send via a MITM proxy such as [Charles](https://www.charlesproxy.com/). 

- Install PreferenceLoader (dependency) and the [Filza](https://filza.net/) (File Browser) on Cydia.
- Download the [latest version from GitHub](https://github.com/nabla-c0d3/ssl-kill-switch2/releases) (deb file).
- Open Filza, navigate to `/private/var/mobile/Library/Mobile Documents/com~apple~CloudDocs/Downloads`.
- Open the downloaded `com.nablac0d3.sslkillswitch2_0.14.deb` (or similar) and press Install.
- Respring. (Restart SpringBoard)
- Find SSLKillSwitch 2 in iOS Settings.

If you're curious how this works on a technical level, here's a [writeup for iOS 12](https://nabla-c0d3.github.io/blog/2019/05/18/ssl-kill-switch-for-ios12/), or just explore the [source on GitHub](https://github.com/nabla-c0d3/ssl-kill-switch2).

# FLEX In-App Debugging

[FLEX](https://github.com/Flipboard/FLEX) is an open-source in-app debugging and exploration tool for iOS by [@NSExceptional](https://twitter.com/NSExceptional). It's amazing what you can do with it. Want the weather background as home screen background? No problem.

{% twitter https://twitter.com/nsexceptional/status/1250353513923674114 %}

To Install, download [FLEXing](http://cydia.saurik.com/package/com.pantsthief.flexing/), reboot and then tap on the status bar to load FLEX. You can browse the classes and inspect the view hierarchy with a 3D debugger, similar to [Reveal](https://revealapp.com/). Here's Spotify:

<img src="/assets/img/2020/jailbreaking/hierarchy-spotify.png" width="80%">

Of course you can also [inspect apps written in SwiftUI](/assets/img/2020/jailbreaking/hierarchy-achelper.png), like the popular [ACHNBrowserUI](https://github.com/Dimillian/ACHNBrowserUI).

# More Useful Cydia Apps & Tweaks

- [iPadify](https://repo.packix.com/package/codes.rambo.ipadify/) — install iPad-only app such as Playgrounds, native picture-in-picture
- [Prysm](https://repo.packix.com/package/com.laughingquoll.prysm/) — a feature-rich control center for iOS
- [Revealloader](https://github.com/heardrwt/RevealLoader) — Load reveal to any app
- [TapTapFlip](https://repo.packix.com/package/com.cpdigitaldarkroom.taptapflip/) — double tap to flip the camera in the Camera app
- [Supercharge](https://www.supercharge.app/) — creating simple tweaks on the device
- [Snapper 2](https://repo.packix.com/package/com.jontelang.snapper2.packix/) — crop screenshots before taking them
- [Frida](https://frida.re/docs/ios/) — a dynamic instrumentation / code injection toolkit
- [FrontCamUnmirror](http://cydia.saurik.com/package/com.sticktron.fcum/) — self explanatory
- [CopyLog](https://repo.packix.com/package/me.tomt000.copylog/) — a powerful clipboard history manager 
- [HomePlus](https://kubadownload.com/news/homeplus-tweak/) — a home screen layout manager
- [FiveIconDock13](https://www.reddit.com/r/jailbreak/comments/e3d3pc/release_fiveicondock13_five_icons_on_your_dock/) — self explanatory
- [Barmoji](https://github.com/CPDigitalDarkroom/Barmoji) and [DockX](https://kubadownload.com/news/dockx-tweak/) — add quick actions below the keyboard

Many tweaks are also open source, which is a great opportunity to learn. Check out [FLEX](https://github.com/Flipboard/FLEX), [Sleeper](https://github.com/joshuaseltzer/Sleeper) (tweaks the stock iOS alarms app), [Open-Source-Tweaks](https://github.com/LacertosusRepo/Open-Source-Tweaks) or the collection at [iPhoneDevWiki](http://iphonedevwiki.net/index.php/Open_Source_Projects).

This is by no means a complete list - see [some](https://twitter.com/iM4CH3T3/status/1264797535316709377?s=20) [inspiration](https://twitter.com/AvimanyuRoy3/status/1264346815165431809) here. Thanks to [everyone who responded to](https://twitter.com/steipete/status/1264893805700026373?s=21) my Tweet to help me collect these gems. Know a tweak I absolutely need to mention? [Hit me up on Twitter](https://twitter.com/steipete)!