---
layout: post
title: "On Using Apple Silicon Mac mini for Continuous Integration"
date: 2020-12-14 11:30:00 +0200
tags: iOS development
image: /assets/img/2020/apple-silicon-ci/trippin.png
description: "Ever since the M1 was announced, we were curious how well Apple's new Mac mini would perform for our CI system. Does it work? Is it worth it? Read and find out."
---

Ever since the M1 was announced, we were curious how well Apple's new Mac mini would perform for our CI system. A few days ago we finally got access to two M1 Mac mini's hosted on MacStadium (8-core M1, 16 GB unified memory, 1TB SSD, 1Gbs).

The Geekbench Score is 1705/7379 vs 1100/5465 so the promise is over 30% more performance, even more so for single-threaded operations Linking, code-signing are all tasks that Apple didn't parallelize yet, so single-core performance is a significant factor for CI performance.

A recap: We run a raw Mac mini setup (6-core 3.2 GHz, 64 GB RAM, 1TB SSD, 10Gbs). I've explored [the tradeoffs between virtualization or bare metal in the PSPDFKit blog](https://pspdfkit.com/blog/2020/managing-macos-hardware-virtualization-or-bare-metal/). 

## Automization Voes

We're using [Cinc](https://cinc.sh/) (the open source binary package of [Chef](https://www.chef.io/products/chef-automate)) + [knife zero](https://knife-zero.github.io/) to automate the setup process for new nodes. It does everything from creating a ci user with an apfs-encrypted drive, configuring firewall rules, installing dependencies like [ccache for faster compiling](https://pspdfkit.com/blog/2020/faster-compilation-with-ccache/) and Ruby for scripting, to installing Xcode and required Simulators. After a few hours, setup is complete and the machine automatically registers itself on Buildkite as new agent.

There's a detailed article coming next in our [series about Continuous Integration for Small iOS/macOS Teams](https://pspdfkit.com/blog/2020/continuous-integration-for-small-ios-macos-teams/) that goes into more detail on this setup. Of course, there have been a few issues along the way to make the automation work with Apple Silicon.

## Installing Rosetta 2

The first thing you'll need to do on the new machines is installing Rosetta to enable Intel emulation. This is curiously not default in Big Sur, but it only takes a few seconds via the terminal:

```
/usr/sbin/softwareupdate --install-rosetta --agree-to-license
```

## Cinc for Darwin/ARM

There's no Chef or [Cinc](https://cinc.sh/start/client/) release yet for Apple Silicon, which is both a blessing and a curse. It does make CI easier, since the Cinc client will run in x64 emulation mode, so anything it installs will also default to x64, which is slower but generally works.

Since the install script does platform detection, it will fail with an error as there's no binary for Cinc yet available. To work around this, modify the `chef_full.erb` script in the [knife](https://github.com/chef/chef/blob/master/lib/chef/knife/bootstrap/templates/chef-full.erb) gem and add `arch -x86_64` before the `sh $tmp_dir/install.sh` part. This will ensure the script detects Intel and will download the client.

Careful: The Chef/Cinc situation is tricky. Don't mindlessly update the gems, as Chef is faster in releasing binaries, overriding all your Cinc binaries and then nothing works anymore unless you insert Dollars or manually remove all Chef gems. There's also a quite messy [randomness between what is renamed "cinc" and what is "chef".](https://twitter.com/steipete/status/1337712935418929154?s=21)

## APFS Containers

We automate diskutil to create a new encrypted volume for the ci user. This ensures that our source code is always encrypted. If hardware is replaced, the data is useless. We manually enter the disk password on a power cycle or when the OS reboots because of an update.

On Apple Silicon, the main APFS container is disk3 and not disk1. Currently this change is hardcoded, eventually I'll modify the the script to parse `diskutil list` to detect the container automatically. It took me quite a while to understand why Cinc stopped with Error: -69493: You can't add any more APFS Volumes to its APFS Container". I mention it here so there's [at least one result on Google with this error.](https://twitter.com/steipete/status/1337711727023157249?s=21) 🙃

## Detecting Apple Silicon via Scripts

We use Buildkite as our CI agent, and they just recently released [3.26.0 with an experimental native executable for Apple Silicon](https://github.com/buildkite/agent/releases/tag/v3.26.0). It's running on a pre-release version of Go, but so far it's been stable.

There is no universal build, so the download script needs adjustment. To not hardcode this, I've been using a trick to detect the *real* architecture at runtime, since the script runs in Rosetta emulation mode and the usual ways would all report Intel.

Here's the full block for Ruby, the interesting part is `sysctl -in sysctl.proc_translated`. It returns `0` if you run on arm, `1` if Rosetta 2, and NOTHING if you run on an Intel-Mac. Everything else is a dance to get the shell output back into Chef-flavored Ruby.

```ruby
action_class do
  def download_url
    #tricky way to load this Chef::Mixin::ShellOut utilities
    Chef::Resource::RubyBlock.send(:include, Chef::Mixin::ShellOut)      
    command = 'sysctl -in sysctl.proc_translated'
    command_out = shell_out(command)
    architecture = command_out.stdout == "" ? 'amd64' : 'arm64'

    platform = ['mac_os_x', 'macos'].include?(node['platform']) ? 'darwin' : 'linux'
    "https://github.com/buildkite/agent/releases/download/v#{new_resource.version}/buildkite-agent-#{platform}-#{architecture}-#{new_resource.version}.tar.gz"
  end
end
```

The best part: This will still work, even if we later switch to Cinc binaries that are native arm.

## Xcode Troubles

I've experimented with using the "Release Candidate" of Xcode 12.3 as main Xcode version, however there's currently [a bug that prevents installing any non-bundled Simulators](https://twitter.com/steipete/status/1337738685282988032?s=21) (we still support iOS 12 in our [iOS PDF SDK](https://pspdfkit.com/pdf-sdk/ios/)), which caused Cinc to stop with an error. For now we stay at Xcode 12.2, hoping that Apple fixes this soon. I assume this is a server-side error so it shouldn't be hard to fix.

There's [a promising fix in Xcode 12.3](https://twitter.com/steipete/status/1336428545791434752) for "improved responsiveness of macOS mouse and keyboard events while under heavy load, such as when building a large project while running Simulator", and a fix for [random lockups of the CoreSimulator service](https://twitter.com/steipete/status/1332348616145563653), so I'm itching to upgrade as soon as possible.

## Test Troubles

Some features in our [iOS PDF SDK](https://pspdfkit.com/pdf-sdk/ios/) use `WKWebView`, like [Reader View that reflows PDFs so they are easier to read on mobile devices](https://pspdfkit.com/pdf-sdk/reader-view/#ios). These tests [crash with a memory allocator error on Big Sur](https://steipete.com/posts/apple-silicon-m1-a-developer-perspective/). 

If you see [`bmalloc::HeapConstants::HeapConstants`](https://gist.github.com/steipete/7181cf321d979d734c5acd2326f6c33f) in a crash stack trace, that's likely this bug. While my radar has no reply yet, I've heard that this is a bug in Big Sur and requires an OS update to be fixed, so this potentially will be resolved in 10.2 some time in Q1 2021.

I've currently worked around this by detecting Rosetta at runtime and then skipping any tests that calls into WebKit using this snippet to detect execution state:

```swift
let NATIVE_EXECUTION = Int32(0)
let EMULATED_EXECUTION = Int32(1)
let UNKNOWN_EXECUTION = -Int32(1)

/// Test if the process runs natively or under Rosetta
/// https://developer.apple.com/forums/thread/652667?answerId=618217022&page=1#622923022
private func processIsTranslated() -> Int32 {
    let key = "sysctl.proc_translated"
    var ret = Int32(0)
    var size: Int = 0
    sysctlbyname(key, nil, &size, nil, 0)
    let result = sysctlbyname(key, &ret, &size, nil, 0)
    if result == -1 {
        if errno == ENOENT {
            return 0
        }
        return -1
    }
    return ret
}
```

## Memory Is Tight

We've been running 6 parallel instances of our tests (one per core) on Intel via the `-parallel-testing-worker-count` option. To make things work well for the M1 chip I reduced the workload to 4 instances. There's really only 4 fast cores, and 4 low-power cores, which perform badly and cause various issues with timeouts in tests. The machine also starts swapping too much memory, as the 16 GB really aren't all that much. Reducing the number to 4 seems to be the best solution to get both more predictable test results and a faster result.

## Breaking Out of Rosetta

Even though our Buildkite agent runs natively, I missed that Cinc installs an Intel-version of Ruby (via `asdf`), and we use Ruby to script CI and parse test results. The Intel-Ruby kicks off `xcodebuild`, which then also runs in emulation mode because we're already in an emulation context. 

I've tried switching to arm-based Ruby. The latest version 2.7.2 does support compiling to arm, but there's still many gems that use native dependencies that haven't been updated yet. Realistically, it will take a while until we can switch to native Ruby there. 

Luckily there's a way to break out: I've been prefixing the `xcodebuild` command with `arch -arm64e` to enforce the native context. This is currently hardcoded in a branch, and I'll use a similar trick to detect the native architecture as in the Ruby script above. Sadly there's no `arch -native` command that would do this for us.

This is important! [Performance is really terrible](https://twitter.com/steipete/status/1338152854662549509?s=21) if Clang runs in Intel-emulation mode.

## More Weirdness

I've encountered a few other weird issues. `launchctl` changed a bit in Big Sur and now throws ["Bootstrap failed: 125: Unknown error: 125"](https://twitter.com/steipete/status/1338155208044638210?s=21) if the service is already running. This again had no Google results, so it took a good bit to understand. Sometimes it would also write "Disk I/O error 5" which caused me to request a complete reset of the machine with MacStadium, only to see the same error many hours later again. In our case the fix was to explicitly unload the Buildkite service before registering it again - this only did show up since the automation script stopped half way due to my various tweaks.

## Results

![Buildkite Test Results](/assets/img/2020/apple-silicon-ci/buildkite.png)

The M1 runs our tests around 10% faster on iOS 14. Tests on older versions of iOS are [around 30-70% slower](https://twitter.com/steipete/status/1338219014338850816?s=21), since the Simulator runs via Rosetta's emulation mode. Results range from 7-9 minutes vs 5 minutes on Intel.

I've also seen [Rosetta bugs in the logs, which caused tests to fail](https://twitter.com/steipete/status/1338152854662549509?s=21). Twitter birds tell me that the Big Sur 11.1 comes with many fixes to Rosetta, so this seems like a transitionary issue.

The machines are marginally cheaper to host (129$/month vs 159$/month on MacStadium) but are still only on limited availability ([we only got 2 even though we ordered 5](https://twitter.com/steipete/status/1337170464460988417?s=21)) and software is still experimental. There's currently more problems than benefits in updating your fleet to M1, especially if you need to support versions below iOS 14.

[My Twitter research](https://twitter.com/steipete/status/1333809139190034433?s=21) thread contains a few more details, and various stages of frustration and delight. Follow me if you enjoy such stories.