---
layout: post
title: UEFI PXE Boot with dnsmasq and proxyDHCP
date: 2024-09-20
description: Setting up a PXE boot server without needing to touch a DHCP server
toc: true
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

### Installing dnsmasq and syslinux-efi

To begin with, update the package lists and install both dnsmasq and syslinux-efi. I am using Debian in this example, where neither of the packages are pre-installed. dnsmasq hosts both the PXE server and proxyDHCP server, whereas syslinux-efi provides the files needed to provide an interface for PXE boot. Stop dnsmasq once installed so that configuration changes can be made safely. This may momentarily affect DNS resolution if using a distribution that relies it on by default - Debian does not so this is fine.

```
sudo apt update
sudo apt install dnsmasq syslinux-efi
sudo systemctl stop dnsmasq
```

### Creating directories

The next step is to create a directory for the files that will be served by the PXE boot server. The directory can be anywhere and named anything, as long as the contents can be read by the dnsmasq user. I named it 'pxeboot'. Another directory called 'pxelinux.cfg' needs to made in this directory, which will contain files required for the PXE boot interface:

```
sudo mkdir /var/lib/pxeboot
sudo mkdir /var/lib/pxeboot/pxelinux.cfg
```

### Copying syslinux files

The following syslinux files need to be copied into the root of the PXE boot directory. These files provide the PXE boot interface as well as the dependencies needed to boot operating system (OS) images:

```
sudo cp /usr/lib/SYSLINUX.EFI/efi64/syslinux.efi /var/lib/pxeboot/
sudo cp /usr/lib/syslinux/modules/efi64/ldlinux.e64 /var/lib/pxeboot/
sudo cp /usr/lib/syslinux/modules/efi64/menu.c32 /var/lib/pxeboot/
sudo cp /usr/lib/syslinux/modules/efi64/libutil.c32 /var/lib/pxeboot/
```

You can also use sym-links should the files ever get updated.

### Configure dnsmasq

Next, dnsmasq needs to be configured so that it acts as a PXE boot server and a proxyDHCP server. The config file is located at:

```
/etc/dnsmasq.conf
```

I prefer to edit files in the terminal with nano:

```
sudo nano /etc/dnsmasq.conf
```

By default, everything is commented out in the config file so everything in it can be removed if desired. The following config can be copied into the file. Comments (#) have been added to explain watch each line does:

```
# PXE config

# Disable DNS server - can be removed if dnsmasq is providing DNS
port=0

# Enable DHCP logging
log-dhcp

# Respond to PXE requests for the specified network + run as DHCP proxy - the network address (10.0.0.0 in this case) may need changing
dhcp-range=10.0.0.0,proxy

# Boot file supplied to clients
dhcp-boot=syslinux.efi

# Provide network boot option called "Network Boot".
pxe-service=x86PC,"Network Boot",pxelinux

# Enable TFTP server for PXE boot
enable-tftp

# PXE boot directory - double check the directory location
tftp-root=/var/lib/pxeboot
```

Whilst dnsmasq is installed, and your system uses an alternate DNS resolver, it will tell your DNS resolver to resolve DNS queries through dnsmasq, essentially giving your system two DNS resolvers. If your system uses a DNS resolver other than dnsmasq, edit the following file and uncomment:

```
sudo nano /etc/default/dnsmasq
DNSMASQ_EXCEPT=lo
```

### Adding OS files and setting up the menu

In it's current state, dnsmasq can serve the syslinux files and clients can boot from them, but the menu doesn't work properly as it hasn't been configured yet. Here is what happens if you PXE boot with this current setup:

![Booting from the PXE server before configuring the menu or any operating systems]({{site.baseurl}}/assets/img/pxe-server/boot.gif)

This is because syslinux needs to be told to create the menu and populate it with OS entries, so you'll need to download an OS. To start with, download an OS which specifically supports network booting - the latest Debian netboot installer can be found at:

[https://d-i.debian.org/daily-images/amd64/daily/netboot/](https://d-i.debian.org/daily-images/amd64/daily/netboot/)

The file needed will be the 'netboot.tar.gz' archive and can be downloaded in the command line with:

```
wget https://d-i.debian.org/daily-images/amd64/daily/netboot/netboot.tar.gz
```

__If you're having DNS issues, it may be because your system uses dnsmasq for resolving DNS queries so you'll need to start dnsmasq up again with:__
```
sudo systemctl restart dnsmasq
```

Next, the downloaded archive can be extracted using:

```
tar -xzf netboot.tar.gz
```

And then the Debian install files can be copied to the pxeboot directory like so:

```
sudo cp -r debian-installer /var/lib/pxeboot
```

The final step is to create the config file that syslinux requires to create the menu and locate OS files. This file needs to be called 'default' and needs to be placed in the 'pxelinux.cfg' directory that was created earlier:

```
sudo nano /var/lib/pxeboot/syslinux.cfg/default
```

In this file, the below config can be pasted which will work with the Debian installer that was just setup. Make sure to remove the comments (#) as the config doesn't work with them in:

```
UI menu.c32
LABEL Debian Net Boot
        MENU LABEL Debian
        KERNEL debian-installer/amd64/linux
                append initrd=debian-installer/amd64/initrd.gz
        TEXT HELP
                Debian installer boot media.
        ENDTEXT
```

In it's current state, the config only has the Debian OS in it. More can be added by simply copying everything except 'UI menu.c32', pasting it on a new line and modifying it to meet the requirements of the new OS that is being added. The config doesn't allow comments in it so heres a breakdown of the config:

``UI menu.c32`` - launches the menu.

``LABEL Debian Net Boot`` - creates a new menu entry, this name does not appear anywhere, it only acts as a reference for you.

``MENU LABEL Debian`` - creates a menu entry with the label "Debian".

``KERNEL debian-installer/amd64/linux`` - this specifies the kernel file to load, PXE boot directory is considered the root directory for syslinux.

``append initrd=debian-installer/amd64/initrd.gz`` - specifies kernel arguments, only one in this case which is specifying the initrd file.

``TEXT HELP`` and ``ENDTEXT`` - mark the beginning and end of the entry description, mulitple lines are allowed between these two directives.

### Test it out!

The last thing left to do is to restart dnsmasq to ensure all configuration changes are applied and test it out:

```
sudo systemctl restart dnsmasq
```

![Debian installer booting from PXE server]({{site.baseurl}}/assets/img/pxe-server/boot_complete.gif)

The Debian installer looks slightly different when booted over netboot. An internet connection is required to complete the installation.

## Conclusion and reflection

I have found this PXE server to be handy when regularly installing operating systems on hardware and virtual machines, it saves time that may be spent looking for an image and then a USB drive to flash it on. It serves as a convenient place to install operating systems from. It isn't without it's faults however, the following are some improvements I would like to look into and implement:

- Multi firmware and architecture support - at the moment, the server only supports UEFI firmware on x86 devices. Ideally, I would like it to at least support legacy BIOS systems too for older machines and operating systems.

- HTTP file transfer - the server currently uses TFTP to transfer files which is rather slow, the Debian initrd file is only 30MB and I'm on a gigabit network so it has the capability of being much faster. I have read that it is possible to implement HTTP for faster file transfers.

- [iPXE](https://ipxe.org/start) - iPXE can be used as an alternative to syslinux. It has the ability to be scripted so that any extra setup can be completed before the OS is booted. I have also read that it may have the ability to boot directly from .iso files which may save hassle when configuring the kernel for boot.

I aim to look into these and may provide an update post in future. Enjoy your PXE server!
