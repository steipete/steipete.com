---
layout: post
title:  "Mac Catalyst — One Year Later"
date:   2020-05-21 10:30:00 +0200
tags: iOS macOS
---

<blockquote class="twitter-tweet"><p lang="en" dir="ltr">Remember this time last year when everyone was going wild over Marzipan rumors?</p>&mdash; Ish (@ishabazz) <a href="https://twitter.com/ishabazz/status/1262060043911827457?ref_src=twsrc%5Etfw">May 17, 2020</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

Worst part: Apple still hasn‘t stated that you just need to use some AppKit to make good Catalyst apps, because that would mean someone inside would have to admit that Catalyst wasn‘t as well thought-out and perfect as they made it out to be. Internal politics at its finest.
Quote Tweet

Still no answer or confirmation if you can use AppKit at all, or how. And while it‘s not very obvious that yes, you can, and Apple does too, this does hurt folks that try to play by the rules/docs. Add in all the unfixed bugs and design mistakes (again, politics) and here we are.

In a sane world, they would have given us an NSUIKitHostingView to wrap views/controllers, but keep AppKit for the shell. Would have created less stress on the UIKit team and opened the door for better apps. Checking the checkbox is a lie.