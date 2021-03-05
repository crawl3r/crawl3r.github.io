---
layout: post
title: "Killing iOS SSL Certificates"





---

## Introduction

A quick introduction to the first step to reading your iOS application traffic. Most applications utilise some form of certificates to sign their traffic with, if they're not... then maybe you should buy the developers a calendar and highlight the current year. Typically, every iOS application pentest I do, killing SSL certificates is the first thing I do and I generally have to set up my test device due to personal derps that happen in between test.

A quick note: I did not write the tool used within this post, however, I have pulled together a quick little bash script to help get the tool deployed to your device.

### SSL Kill Switch 2

The best tweak I have come across for achieving this task is https://github.com/nabla-c0d3/ssl-kill-switch2. Once installed, you can utilise it's settings window to kill all certificates across the device as well as specify certain app ID's to ignore. Please be warned, that installing this tweak to your device leaves your traffic open to be sniffed by well positioned attackers, but you have already ran an exploit to root your phone... so, I guess we have already accepted that risk.

If you would like to know a bit more about this tool, check out the git linked above and check out the authors blog: https://nabla-c0d3.github.io/blog/2019/05/18/ssl-kill-switch-for-ios12/

### Set Up Script

As previously mentioned, the quick script I utilise to deploy this tool whenever required can be found below. It's nothing crazy, doesn't exactly do anything that you can't do with a few commands, however, if I can save 5 seconds on a job... it means I can do something else, like sip coffee /s.

```bash
# Note: This expects wget + dpkg to be installed
# Dependencies for killswitch are:- Debian Packager, Cydia Substrate, PreferenceLoader

# The URL here is linked to the latest 'release'. Manually update this if there is a newer version available
wget -O /Users/Downloads/killswitch.deb https://github.com/nabla-c0d3/ssl-kill-switch2/releases/download/0.14/com.nablac0d3.sslkillswitch2_0.14.deb

# install the new debian package
dpkg -i /Users/Downloads/killswitch.deb

# Restart springboard
killall -HUP SpringBoard

# Delete the downloaded file
rm -rf /Users/Downloads/killswitch.deb
```

