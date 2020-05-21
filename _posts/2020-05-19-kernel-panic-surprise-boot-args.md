---
layout: post
title:  "Kernel Panics and Surprise boot-args"
date:   2020-05-21 10:00:00 +0200
tags: personal
---

Starting March 24, 2020, my 16-inch MacBook Pro is greeting me with a kernel panic almost every morning.

<blockquote class="twitter-tweet"><p lang="en" dir="ltr">Regression: MacBook Pro 16-inch panics almost every night in AppleIntelFramebuffer::setPowerState. This started with macOS 10.15.4 - FB7642937 is someone cares.</p>&mdash; Peter Steinberger (@steipete) <a href="https://twitter.com/steipete/status/1243854244115091456?ref_src=twsrc%5Etfw">March 28, 2020</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

Was it just me? [No](https://twitter.com/jernejv/status/1243854771905273857?s=20), [seems](https://twitter.com/SergejBerisaj/status/1243857963724558337?s=20) [this](https://twitter.com/AlexManzer/status/1244606008955146240?s=20) [is](https://twitter.com/lostincode/status/1243900563902717953?s=20) [really](https://twitter.com/collinluke/status/1251668176296910849?s=20) [fairly](https://twitter.com/pagetable/status/1244599318151155712?s=20) [wide](https://twitter.com/BarrosMyles/status/1244021525562474497?s=20) [spread](https://twitter.com/slaven/status/1244532699139731456?s=20). ([Even on the MacBook Air 2018](https://twitter.com/AVMatiushkin/status/1249671960713482240?s=20))

Great. Rebooting every is annoying, so I wrote a r̶a̶d̶a̶r̶ Feedback Assistant entry: FB7642937.

>Regression: MacBook Pro 16-inch panics almost every night in AppleIntelFramebuffer::setPowerState. This started with macOS 10.15.4

(Some reports indicate that this happened with 10.15.3 earlier, but for me it started with 10.15.4)

The backtrace surprisingly readable and indicates a timeout in Intel's graphic driver (AppleIntelFramebuffer):

```
panic(cpu 2 caller 0xffffff8014016487): "AppleIntelFramebuffer::setPowerState(0xffffff835c3b6000 : 0xffffff7f975f5d88, 1 -> 0) timed out after 45938 ms"@/AppleInternal/BuildRoot/Library/Caches/com.apple.xbs/Sources/xnu/xnu-6153.101.6/iokit/Kernel/IOServicePM.cpp:5296
Backtrace (CPU 2), Frame : Return Address
0xffffff83be6abb40 : 0xffffff80139215cd 
0xffffff83be6abb90 : 0xffffff8013a5a3c5 
(...)

BSD process name corresponding to current thread: kernel_task
Boot args: rootless=0 kext-dev-mode=1 agc=2 smc=0x2 watchdog=0 nvme=0x9 legacy_hda_tools_support=1 sandcastle=0 amfi_hsp_disable=1 chunklist-security-epoch=0 -chunklist-no-rev2-dev
```

# fun boot-args
I've posted this on Twitter to learn more, but while nobody could tell me what's up here, a friend noticed that I have ["fun boot-args"](https://twitter.com/NSBiscuit/status/1243294676985294849?s=20). WTH? 

I didn't set any of these, but I had to send my 16-inch MacBook in for repairs, as it corrupted memory, resulting in [extremely](https://twitter.com/steipete/status/1230925689098002433) [weird bugs](https://twitter.com/jckarter/status/1230253181495459841) and [frequent kernel panics](https://twitter.com/gparker/status/1231155681991909376). They replaced most of the machine, so it came with a fresh install of macOS... and some interesting boot-args that the engineers there forgot to clear. (If you're curious what these args do, some are [documented here](https://superuser.com/questions/255176/is-there-a-list-of-available-boot-args-for-darwin-os-x), and some more if you look towards the Hackintosh scene)

## Are these dangerous?

Having my main computer run with unknown boot-args did worry me. What did they do? I went on a google-spree to find out what all of these do, one by one.

[`rootless=0`](https://www.cryptomonkeys.com/2015/07/osx-rootless-boot-args/) Rootless (a security mechanism making its debut in El Capitan 10.11) restricts access to certain files, even as an administrative user, such as is provided by ‘sudo’. 

[`kext-dev-mode=1`](https://apple.stackexchange.com/questions/311065/what-does-setting-boot-args-kext-dev-mode-do-to-set-the-serial-port) is intended for development scenarios where you want to test unsigned kernel extensions. Thus allowing these to be loaded without testing if they are indeed signed.

[`agc=2`](https://gist.github.com/blackgate/17ac402e35d2f7e0f1c9708db3dc7a44). This seems related to automatic graphic switching between Intel and AMD Radeon GPU drivers. The setting 2 seems to prevent the AMD chip from powering down. Information around this is shaky.

`smc=0x2` SMC is short for [system management controller](https://support.apple.com/en-us/HT201295), responsible for battery, fans, sensors and thermal management. While I found some reports with this flag, I couldn't figure out what this does.

[`watchdog=0`](http://www.hari.xyz/2019/01/setting-up-os-x-for-kernel-debugging.html) disables macOS watchdog from firing when returning from the kernel debugger.

[`nvme=0x9`](https://pikeralpha.wordpress.com/2016/06/15/nvme-boot-argument/) This controls the Non-Volatile Memory Express controller, and might disable the ‘Apple SSD’ check. Couldn't find info on 0x9.

[`legacy_hda_tools_support=1`](https://github.com/acidanthera/AppleALC/blob/master/AppleALC/kern_alc.cpp). Unlocks custom audio engines by disabling Apple private entitlement verification.

`sandcastle=0`. It is definitely not [Project Sandcastle, enabling Android to run on iPhones](https://arstechnica.com/gadgets/2020/03/project-sandcastle-brings-android-to-the-iphone/). It sounds security related, but I couldn't find details.

[`chunklist-security-epoch=0 -chunklist-no-rev2-dev`](https://gist.github.com/devzer01/e24dc78150d574ade3382eaddaf1827a) I link them together because they sound related, and often show up in combination in random kernel panic reports. No idea what they do, but it again seems to disable something related to security.

Having a long list of undocumented boot-args running on my machine for weeks that seem to weaken security is definitely worrying, and not what you expect from a repair at an Apple Store.

Clearing boot args only works in Recovery Mode (⌘-R at boot time), unless SIP is disabled.
```
sudo nvram boot-args=""
```

# Conclusion

Turns out, the boot-args were not related to the kernel panics, clearing them changed nothing related to the panic rate, but I'm glad I found and cleaned this.

Will Apple reply to my report? To be continued...