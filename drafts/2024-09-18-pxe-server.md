---
layout: post
title: UEFI PXE Boot with dnsmasq and proxyDHCP
date: {}
description: Setting up a PXE boot server without needing to touch a DHCP server
tags:
  - services
---
## Introduction

Typically, setting up a PXE boot server requires a DHCP server that allows you to configure DHCP options [66](https://www.rfc-editor.org/rfc/rfc2132.html#section-9.4) and [67](https://www.rfc-editor.org/rfc/rfc2132.html#section-9.5), which tell clients the address of the PXE boot server and the name of the file they are to request and boot from. However, this functionality is not available on most consumer grade equipment without modification (see OpenWrt). Using [dnsmasq](https://dnsmasq.org/doc.html) to host a proxyDHCP server circumvents this by supplying additional DHCP packets including PXE boot information to clients which request it. dnsmasq can also be used as a PXE boot server, making this an all-in-one solution.

## Why would you want to do this?

Rather than replacing your ISP router with dedicated equipment that can do the job, you may rather retain a setup supported by your ISP in order to avoid any funny business when requesting support from them. As well as this, you might not want to run a dedicated DHCP server separate from the one you already have running in your router, as this can introduce an additional point of failure to the network.

## How it works

![Wireshark capture of the DHCP process]({{site.baseurl}}/assets/img/pxe-server/pxe_wireshark.png)

### Discover

The process starts as normal, with the client broadcasting a DHCP Discover packet requesting DHCP options 66 and 67, amongst others (packet 958):

![DHCP option requests from the client]({{site.baseurl}}/assets/img/pxe-server/client_discover_requests.png)

For comparison with a device not wanting to PXE boot, here is what my Pixel 7 requests:

![DHCP option requests from my Pixel 7]({{site.baseurl}}/assets/img/pxe-server/pixel7_discover_requests.png)

### Offer

Both the DHCP server and the proxyDHCP server broadcast a DHCP Offer packet in response. The packet from the proxyDHCP server supplies the IP address of the PXE server (itself), as well as the name of file the client needs to request:

![DHCP offer from proxyDHCP]({{site.baseurl}}/assets/img/pxe-server/proxy_offer.png)

The packet from the DHCP server only supplies an IP address for the client, as well other standard IP addressing information:

![DHCP offer from the DHCP server]({{site.baseurl}}/assets/img/pxe-server/dhcp_offer.png)

### Request and Ack with DHCP server

The client then sends a standard request packet to the DHCP server, requesting the IP address it was offered, to which it receives a standard acknowledgment:

![DHCP request and ack packets between client and DHCP server]({{site.baseurl}}/assets/img/pxe-server/client_dhcp_server_request_and_ack.png)

### proxyDHCP Request and Ack

Once the client has an IP address, it then responds to the proxyDHCP offer from earlier to request the PXE data the DHCP server wasn't able to provide. In this request, the client provides the same list of requested options as it did in it's initial Discover packet:

![proxyDHCP request sent directly to the proxyDHCP server]({{site.baseurl}}/assets/img/pxe-server/proxy_request.png)

The proxyDHCP server acknowledges this request by first informing the DHCP server of the packet it is sending to the DHCP client, and then sends the acknowledgment to the client, with the source MAC address set as that of the DHCP server, so that the client understands it to be a continuation of the information already supplied by the DHCP server:

![proxyDHCP ack packets to the DHCP server and client]({{site.baseurl}}/assets/img/pxe-server/proxy_acks.png)

The DHCP client can then request the boot file from the PXE boot server via TFTP.

## Installing and configuring

To begin with, update the package lists and install both dnsmasq and syslinux-efi. I am using Debian in this example, where neither of the packages are pre-installed. dnsmasq hosts both the PXE server and proxyDHCP server, whereas syslinux-efi provides the files needed to provide an interface for PXE boot. Stop dnsmasq once installed so that configuration changes can be made safely. This may effect DNS resolution if using a distribution that relies it on by default - Debian does not so this is fine.

```
sudo apt update
sudo apt install dnsmasq syslinux-efi
sudo systemctl stop dnsmasq
```

![Installing dnsmasq and syslinux-efi]({{site.baseurl}}/assets/img/pxe-server/update_and_install.png)
