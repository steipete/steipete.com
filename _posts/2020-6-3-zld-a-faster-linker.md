---
layout: post
title:  "zld - a faster version of Apple's linker"
date:   2020-06-03 14:00:00 +0200
tags: iOS development
---

zld is [a fork of Apple's linker](https://github.com/michaeleisel/zld#a-faster-version-of-apples-linker) that promises to be at least 40% faster than Apple's default ld. It comes with a great promise:  

> “Feel free to file an issue if you find it's not at least 40% faster for your case”

In our setup, this indeed improves overall build time by ~25% (from a clean build to the running binary).

Building PSPDFCatalog in debug mode with [ccache](https://pspdfkit.com/blog/2015/ccache-for-fun-and-profit/) enabled and everything pre-cached:

- ld: 4:40min
- zld: 3:30min

Here's how you can enable zld:

```
// Settings to improve link time performance for debug builds
OTHER_LDFLAGS = -ObjC -Wl,-no_uuid, -fuse-ld=/usr/local/bin/zld
```

For our setup at PSPDFKit I wrote a conditional wrapper, so building works no matter if you have this installed or not. This is called `xld-detect` and stored [in our monorepo](https://pspdfkit.com/blog/2019/benefits-of-a-monorepo/).

```
#!/bin/sh

# /usr/local/bin is not always included in the Xcode context
export PATH="$PATH:/usr/local/bin"

# Detect if zld is available
if type -p zld >/dev/null 2>&1; then
  exec zld "$@"
else
  exec ld "$@"
fi
```

You can call this script directly from your Xcode project(s):

```
OTHER_LDFLAGS = (
	"$(inherited)",
	"-fuse-ld\"=$(SRCROOT)/../Resources/zld-detect\"",
);
```

That's it! Let me know on Twitter if this was helpful.