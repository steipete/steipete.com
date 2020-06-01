---
layout: post
title:  "Updating macOS on a Hackintosh"
date:   2020-06-01 10:00:00 +0200
tags: hardware
image: /assets/img/2020/hackintosh/opencore-config.png
---

With macOS 10.15.5 out, it was time to update my Hackintosh again. This does take a bit more preparation than updating genuine Mac hardware. Since a few folks on Twitter were curious, here's how this works.

## Background

Earlier this year I built myself a Hackintosh, [similar to this setup](https://github.com/cmer/gigabyte-z390-aorus-master-hackintosh) ([picture porn here](https://infinitediaries.net/my-2020-hackintosh-hardware-spec/)). I still work 95% of my time on my 16-inch MacBook Pro, but it's convenient to have a really powerful machine to compile large projects (like Swift). I'm impressed with the overall stability - I had considerably less [kernel panics](/posts/how-to-macos-core-dump/) here than with my MacBook Pro, and I'm using the LG 5K in full resolution on the Hackintosh, thanks to the [Gigabyte GC-Titan Ridge](https://www.amazon.com/GIGABYTE-GC-Titan-Ridge-Thunderbolt-Component/dp/B07GBZL93X).

To run macOS on non-Apple-hardware, you need a bootloader. I chose [OpenCore](https://github.com/acidanthera/OpenCorePkg), as it's fast, modern and open source. Legacy setups might still use [Clover](https://github.com/CloverHackyColor/CloverBootloader), but OpenCore is much better long-term, and you don't have to disable security features such as SIP. If you're curious about building a Hackintosh, check out the [OpenCore Vanilla Guide](https://khronokernel-2.gitbook.io/opencore-vanilla-desktop-guide/).

## Bootable USB Drive

Before updating macOS, we want to update OpenCore and the needed kexts. You can do this on your machine directly, but it might not boot anymore if you make a mistake. It's better to first test-drive the new boot loader on a removable drive:

- Get a USB drive and GUID format it to HFS+ Journaled.
- Mount both your boot disk EFI and your USB drive EFI partitions via [MountEFI](https://github.com/corpnewt/MountEFI).
- Copy the `EFI` folder from your boot drive EFI partition to your USB drive EFI partition.\
(I know, so much EFI!)
- Reboot, select the USB drive as source in your bios, to verify that you can boot from the disk.

## Updating OpenCore

Did it work? Great, now let's update OpenCore! There's a monthly release cycle not only for OpenCore but also the core kexts, so it's smart to wait a few days to update.

- Get the [latest OpenCore release](https://github.com/acidanthera/OpenCorePkg/releases). I cheated a bit and got me a nightly build of [0.5.9-pre](https://github.com/williambj1/OpenCore-Factory/releases), since the release is just a few days away and it seemed worth the risk.
- Replace `EFI/BOOT/BOOTx64.efi`, `EFI/OC/OpenCore.efi` and `EFI/OC/Drivers/OpenRuntime`.

If you never done this before, I recommend reading [Updating OpenCore and macOS](https://dortania.github.io/OpenCore-Desktop-Guide/post-install/update.html) - this explains the steps in far more detail. 

## Updating config.plist

Now the fun part starts! OpenCore is configured with one giant `config.plist` and is really picky about it. You want to be sure to get this right, also the format changes slightly with every release.

Use [OCConfigCompare](https://github.com/corpnewt/OCConfigCompare) to compare between the `sample.plist` and your `config.plist` The script has an auto-download feature, but since I used a nightly here, I set a manual folder.

Use Xcode to edit the plists files, and be meticulous. Don't forget that your config contains different settings, so be on the lookout if a key has simply be renamed and migrate your values. 

![OpenCore Config](/assets/img/2020/hackintosh/opencore-config.png)

Done? Good. [Upload the file here](https://opencore.slowgeek.com/) for a sanity check and go back and fix the parts you missed :)

## Updating Kexts

You will have a variety of kexts to support your Hackintosh hardware. Here are mine:

- [AppleALC](https://github.com/acidanthera/applealc/releases)
- [IntelMausi](https://github.com/acidanthera/intelmausi/releases)
- [Lilu](https://github.com/acidanthera/lilu/releases)
- [VirtualSMC](https://github.com/acidanthera/virtualsmc/releases)
- [WhateverGreen](https://github.com/acidanthera/whatevergreen/releases)

Update as needed, then boot from the updated EFI on your USB drive. If everything works, copy the new EFI folder back to your boot drive EFI partition. You can unplug the drive now, it's no longer needed.

## Update Drivers & Tools

OpenCore also comes with efi-drivers and tools, and you can see if there's anything new or interesting. Make sure to delete outdated drivers. I updated from 0.5.6 to 0.5.9 so a few drivers were merged into OpenRuntime.

I used `VBoxHfs-64.efi` and replaced it with `HFSPlus.efi`. The former is the HFS+ driver from VirtualBox, the latter is Apple's original driver, which is supposedly faster.

## Update Time 🥁

Now it's time to update! I don't use my Hackintosh as primary system, and do not recommend it, but if you do, please make a backup (I recommend [SuperDuper!](https://www.shirt-pocket.com/SuperDuper/SuperDuperDescription.html) and [Arq](https://www.arqbackup.com/)).

There's no guarantee your system will come backup again, however it did in my case, and the whole update procedure - including writing this post - took me about 1.5 hours. Not too bad, eh?