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

Thanks to the community surrounding this device on XDA Forums, there is an open-source kernel available for it, which means that a customised kernel can be built specifically for Nethunter to run on this device, in theory.

This blog post aims to cover my journey with this project as not only is this process rather new to me, I also found that the offical documentation for this process was rather vague

## Setting up

