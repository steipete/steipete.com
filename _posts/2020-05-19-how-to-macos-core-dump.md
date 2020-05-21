---
layout: post
title:  "How to macOS Core Dump"
date:   2020-05-21 10:30:00 +0200
tags: personal
---

If you just started reading now, check out the backstory: [Kernel Panics and Surprise boot-args](https://steipete.com/posts/kernel-panic-surprise-boot-args/).

A few weeks in (and many comments where I kept pinging Apple and reporting on this issue, trying a Supplemental Update but still suffering from this bug), somebody recommended to me that I should disable the ["Power Nap"](https://support.apple.com/en-gb/HT204032) feature. I was skeptical, but sure enough, ever since I disabled it, it never happened again. 

I bruised it off and almost forgot about it, but two months after I've reported my issue via Feedback Assistant, Apple indeed replied to my Feedback!
 
![Apple Feedback](/assets/img/2020/appleintelframebuffer/feedback.png)

I mean, it's something. But it's also somewhat of a joke. After my initial reaction of getting mad how bad Apple's communication here, I started seeing it as a challenge and started my search as to what they could have possibly meant with a "Core Dump".

# The great Core Dump mystery

If you google "macOS core dump" [you will eventually land on StackOverflow](https://stackoverflow.com/questions/9412156/how-to-generate-core-dumps-in-mac-os-x/12118329 ), as with so many things.

The article recommends setting `ulimit -c unlimited`, but this didn't seem right - how's that related to a kernel panic? Next up I looked into Apple's [Technical Note TN2151:
Understanding and Analyzing Application Crash Reports](ttps://developer.apple.com/library/archive/technotes/tn2124/_index.html#//apple_ref/doc/uid/DTS10003391-CH1-SECCOREDUMPS), which has a section about Core Dumps, and also mentioned `ulimit`.

>You can enable core dumps on a system-wide basis by adding the line limit core unlimited to your /etc/launchd.conf file

Problem: This file doesn't even exist on macOS Catalina. The document has been last updated in 2010, so that might explain why.

Via [Quinn “The Eskimo!”](https://twitter.com/justkwin/status/1260674056023572481) I also learned about [the use case for application core dumps to debug crashers](https://forums.developer.apple.com/message/401103#401103).


# Kernel Core Dump (lings)

I've asked around on Twitter and learned that [a *kernel* core dump is something else entirely](https://twitter.com/gparker/status/1260714803191885825). Thanks to [Sarah](https://twitter.com/winocm/status/1260815939001446406?s=21) for helping me here!

Welcome to 2-easy-steps to create your kernel core dump:

1: Change the boot-args to enable kernel debugging. This needs to be run with csrutil disable or in recovery mode.
`keepsyms=1` is used to symbolicate the panic log.

```
sudo nvram boot-args="debug=0x104c44 keepsyms=1"
```

2: Once you get a panic with the debug= boot-args set, just dd the `Apple_KernelCoreDump` volume on your APFS container to a file:

```
dd if=“~/IntelFramebufferFun.img” of="/Volumes/Apple_KernelCoreDump" 
```

# Deciphering 0x104c44

Of course my curiosity sparked when reading `debug=0x104c44`, which leads to just one search result for a [crash with VirtualBox](https://forums.virtualbox.org/viewtopic.php?f=8&t=92617). 

![Wisdom of the Ancients, XKCD 979](https://imgs.xkcd.com/comics/wisdom_of_the_ancients.png)

Al of these flags are related to macOS Kernel Debugging, so here's a great opportunity to learn!

`0x104c44 = 1 0000 0100 1100 0100 0100`

`DB_NMI 0x4` changes the power-button to create a non-maskable interrupt? (These days, NMI is ctrl+cmd+alt+shift+power, old behaviour is via 0x8000)

`DB_ARP 0x40` allows debugging across subnets via ARP (usual flag)
 
`DB_KERN_DUMP_ON_PANIC 0x400` Trigger core dump on panic

`DB_KERN_DUMP_ON_NMI 0x800` Trigger core dump on NMI

`DB_REBOOT_POST_CORE 0x4000` Attempt to reboot after post-panic crashdump/paniclog dump

`DB_REBOOT_ALWAYS 0x100000` Don't wait for debugger connection

Let's check if we got everything: `0x100000 + 0x4000 + 0x800 + 0x400 + 0x40 + 0x4 = 0x104c44`

## Articles that helped
- [2008: Technical Note TN2118:Kernel Core Dumps](https://developer.apple.com/library/archive/technotes/tn2004/tn2118.html)
- [2018: Scott Knight, macOS Kernel Debugging](https://knight.sc/debugging/2018/08/15/macos-kernel-debugging.html)
- [2019: GeoSn0w, Debugging macOS Kernel For Fun](https://geosn0w.github.io/Debugging-macOS-Kernel-For-Fun/)

# Conclusion
After writing up my story here, I found [an article on "Mr. Macintosh"](https://mrmacintosh.com/10-15-4-update-wake-from-sleep-kernel-panic-in-16-mbpro-2019/) which documents the very same wake-from-sleep kernel panic. It seems it is fixed in macOS 10.15.5 Beta 4, so there's no need for me to actually send a macOS Core Dump anymore.

Still, it was an interesting road and learning experience.

# Update: Human Reply + coredump via Network
A week after their initial "macOS Core Dump" reply, and me sending a lot of questions their way, I got a really nice reply that explains the process via networking and a second Mac, so I'm sharing this fore future reference:

Any Mac will work as a coredump server; you just need a gigabyte or so of free space per coredump. The kernel dump client can only be configured to transmit on a hard-wired Ethernet port, either built-in or over Thunderbolt. There is no support for transmitting core dumps across the AirPort interface, USB Ethernet, or across third-party Ethernet interfaces. This is an issue for early MacBook Air models which have no built-in Ethernet or Thunderbolt interfaces.

On the server (non-panicking) machine run:
- `sudo mkdir /PanicDumps`
- `sudo chown root:wheel /PanicDumps`
- `sudo chmod 1777 /PanicDumps`
- `sudo launchctl load -w /System/Library/LaunchDaemons/com.apple.kdumpd.plist`

To verify that the core dump server is active
`sudo launchctl list | grep kdump`  
This should return: `- 0 com.apple.kdumpd`

On the client (panicking) machine run the following commands
Locate the IP address of the core dump server.  
`sudo nvram boot-args="debug=0xd44 _panicd_ip=10.0.40.2 kdp_match_name=en7”`

Where `10.0.40.2` is replaced by the IP address of the server and en7 is replaced by the name of the client’s Ethernet interface. You can use this command to show all of the network interfaces on the system: `ifconfig -a`

Then reboot: `sudo reboot`

If you hang, NMI the machine by hitting the buttons Left-⌘ + Right-⌘ + Power. This will generate a coredump file on the server machine in the directory `/PanicDumps`. If you panic, the coredump file will be generated automatically on the server machine in the `/PanicDumps` directory. Compress the coredump file saved to `/PanicDumps`, and attach that to a feedback report. (Attachments have to be a zip, not a folder)



