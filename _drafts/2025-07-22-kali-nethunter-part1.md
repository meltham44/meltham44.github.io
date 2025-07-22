---
layout: post
title: Building Kali Nethunter for an unsupported device (part 1)
date: {}
description: Getting started with kernel building for the Wileyfox Swift 2 X
toc: true
---
## Introduction

Kali Nethunter is an Android-based penetration testing platform built for mobile devices and is based on the Kali Linux desktop operating system. It provides tools, functions and drivers similar to those found in the full desktop experience, all with the benefit of running them on a mobile device.

Kali Nethunter is available in three different editions, with each subsequent one capable of doing more than the previous one. These are:

- Nethunter Rootless (limited to local programs and tools)
- Nethunter Lite (additional tools and services with the Nethunter app)
- Nethunter (the full experience featuring additional drivers for USB gadgets and network devices as well as Wi-Fi injection support)

The full Nethunter experience requires a custom kernel to be built for the target device which features the required additional drivers and patches. There are some supported devices which already have the full Nethunter experience built for them, however the device I own and wish to install Nethunter on isn't one of them.

The aim of this project is to build the complete Nethunter experience for a Wileyfox Swift 2 X (codename marmite). This curious little device was my daily driver from 2018 to 2021 and I ran LineageOS 16.0 on it for most of it's life.

// picture of phone

Thanks to the community surrounding this device on [XDA Forums](https://xdaforums.com/f/wileyfox-swift-2-roms-kernels-recoveries-othe.6039/), there are open-source kernels available for it, which means that a customised kernel can be built specifically for Nethunter to run on this device, in theory.

This blog post aims to cover my journey with this project as not only is this process rather new to me, I also found that the offical documentation for this process was rather vague. I am aiming to do these blog posts in parts as I manage to get parts of this project working.

## Setting up

Before starting on the build, I needed a custom ROM installed on my device which had a readily available kernel source. This was so that once the kernel was built, it could be installed in-place of the device's current kernel and tested to ensure it was built correctly by seeing if it booted into it's installed ROM as expected. My Swift 2 X was already running [nikith290's build](https://xdaforums.com/t/rom-unofficial-pie-lineageos-16-0-for-wileyfox-swift-2-plus-x.3906452/) of LineageOS 16.0 (based on Android 9/Pie), however I chose to flash it with an upgrade to [robin0800's build](https://xdaforums.com/t/lineage-18-1-20220306-unofficial-marmite.4412121/post-90147704) of LineageOS 18.1 (based on Android 11). I'm not sure if this would have made any difference, and I probably could have kept using the kernel and ROM made for LineageOS 16.0 (they're both based on Linux kernel 3.18), however I think using the newer build and kernel meant that I could build Nethunter for Android versions up to 11. I also needed [TWRP](https://twrp.me/about/) installed on my device, which was already installed from initally flashing LineageOS, and I needed the [Android SDK Platform Tools](https://developer.android.com/tools/releases/platform-tools) to interact with my device over the command line.

To begin with, I launched TWRP and reformatted my device's data partition so that it could be seen and mounted within TWRP.

// formatting data

Next, I downloaded the zip file for LineageOS 18.1 and pushed it to the device via ADB and proceeded to install it.

// pushing zip via adb and installing

Once this was done, I rebooted the device and successfully booted into LineageOS 18.1.

// New ROM

The next step was to clone the source code for this ROM's kernel, which could be found within the original post 