---
layout: post
title: "Comparing Update Times of Windows and macOS"
date: 2020-08-12 10:00:00 +0200
tags: development
image: /assets/img/2020/os-updates/windows-stats.png
---

<style type="text/css">
div.post-content > img:first-child { display:none; }
</style>

At [PSPDFKit](https://pspdfkit.com/) we ship PDF SDKs for all major operating systems, including macOS and Windows. We regularly look at update rates to see when it's time to drop older versions. Today I noticed something interesting: Windows now updates much faster than macOS.

## Windows Updates over Time

Historically, updating Windows was a big problem. Here's the distribution share for the last ~10 years of Windows, and some of the key releases:

- Oct 2001: Windows XP
- Jan 2007: Windows Vista
- Oct 2009: Windows 7
- Oct 2012: Windows 8
- Oct 2013: Windows 8.1
- Jul 2015: Windows 10

I'm skipping Server-versions as they aren't statistically significant for this post. If your business is mostly B2B (as we are) this might change the statistics to a small degree.

[![Windows Version Market Share Jan 2009-July 2020](/assets/img/2020/os-updates/windows-stats.png)](https://gs.statcounter.com/windows-version-market-share/desktop/worldwide/#monthly-200901-202007)

The only significant releases (over 30% market share) were Windows XP, Windows 7 and now Windows 10. Most consumers simply don't update their machines, don't wanna bother or are afraid that something might break. Updating the OS generally happens by purchasing a new computer. For a time, Windows Vista and Windows 8 had such a bad reputation that OEMs offered new computers with previous versions of Windows.

It took *10 years* for Windows XP to fall below 50% market share. For Windows 10, it's been on over 50% of machines in *just about 3 years*. So, what changed?

## Windows 10 Updates

Microsoft did something really clever with Windows 10 - not only did they give away the update for free, they also marketed it as ['the last version of Windows'](https://www.theverge.com/2015/5/7/8568473/windows-10-last-version-of-windows).

Instead of new major releases every ~3 years, Microsoft switched to features updates. These are technically new versions of the OS, which are available twice a year, during spring and fall time frame. Microsoft calls this "Windows as a Service (WaaS)"[^1]

Feature updates install automatically and Microsoft's been really careful not to break older software - so most users might not even notice the difference. There's also "quality updates" ("cumulative updates") that come once a month. Customers can delay an update, but it's not easy to block one completely. See also: [Windows lifecycle fact sheet](https://support.microsoft.com/en-us/help/13853/windows-lifecycle-fact-sheet)

[^1]: For Server-customers, Microsoft still does release names, after Windows Server 2016 (based on Windows 10 Build 1607), Windows Server 2019 (based on Windows 10 Build 1809) there will likely be a Windows Server 2022.

## Comparing Windows and macOS Update Rates

Let's look at the latest numbers and assume that [73% of all PCs](https://gs.statcounter.com/windows-version-market-share/desktop/worldwide/#monthly-200901-202007) run Windows 10 (Windows 7 is at ~20%).

| OS                      | Date         | Market Share |
|-------------------------|--------------|--------|
| Windows 10 2004         | May 27, 2020 | 11.60% |
| Windows 10 1909         | Nov 12, 2019 | 36.80% |
| Windows 10 1903         | May 21, 2019 | 43.60% |
| Windows 10 1809         | Nov 13 2018  | 3%     |
| Windows 10 1803         | Apr 30 2018  | 2.50%  |
| Windows 10 1709         | Oct 17 2017  | 0.80%  |

Over 67% of all computers run a Windows 10 release from 2019, vs 55% for macOS. 

| OS                      | Date         | Market Share |
|-------------------------|--------------|--------|
| macOS 10.15 Catalina    | Oct 4, 2019  | 54.94% |
| macOS 10.14 Mojave      | Sep 24 2018  | 17.72% |
| macOS 10.13 High Sierra | Sep 25 2017  | 13.25% |
| macOS 10.12 Sierra      | Sep 20 2016  | 5.24%  |

Microsoft is already faster with distributing update, and over time this gap will widen - the "Windows as a Service" strategy seems to work.

## Sources

There's no official statistics from either Microsoft or Apple about install numbers, so I'm using 3rd-parties that get their data by tracking what browsers report.

* [Desktop Windows Version Market Share](https://gs.statcounter.com/windows-version-market-share/desktop/worldwide/#monthly-200901-202007)
* [Windows 10 OS Worldwide](https://reports.adduplex.com/#/r/2020-07)

