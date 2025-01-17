---
title: Legacy proxy dhcp configuration
description: Configuration of a proxy dhcp service like dnsmasq to use fog as a
  pxeboot server, this is the legacy version of this doc, it may need to be
  removed or merged with the newer version of the doc after they're both
  reviewed
aliases:
    - Legacy version of Proxy DHCP with dnsmasq
    - Using FOG with an unmodifiable DHCP server (Legacy)
    - Legacy proxy dhcp configuration
tags:
    - pxe
    - ipxe
    - dhcp
    - proxy
    - proxy-dhcp
    - option-66
    - option-67
    - advanced-configuration
    - network
    - network-config
    - legacy
---


# Legacy proxy dhcp configuration

> [!note]
> This article is quality and may be followed, however a new article is written at the below link that includes UEFI support.
> [[proxy-dhcp|Proxy DHCP using DNSMasq]]


## Overview

This combines FOG with a proxyDHCP server. What a proxyDHCP service does
is listen to DHCP requests and respond to clients identifying themselves
as PXE Clients. It leaves the role of assigning IP addresses to the
other DHCP servers, but provides the necessary information so the client
can PXE boot. ProxyDHCP is a solution for those of you who are working
with an unmodifiable DHCP server or wish to avoid the hassle of editing
the already existing DHCP server, or even as a portable imaging
solution.

## How ProxyDHCP works

1.  When a PXE client boots up, it sends a DHCP Discover broadcast on
    the network, which includes a list of information the client would
    like from the DHCP server, and some information identifying itself
    as a PXE capable device.
2.  A regular DHCP server responds with a DHCP Offer, which contains
    possible values for network settings requested by the client.Usually
    a possible IP address, subnet mask, router (gateway) address, dns
    domain name, etc.
3.  Because the client identified itself as a PXEClient, the proxyDHCP
    server also responds with a DHCP Offer with additional information,
    but not IP address info. It leaves the IP address assigning to the
    regular DHCP server. The proxyDHCP server provides the
    next-server-name and boot file name values, which is used by the
    client during the upcoming TFTP transaction.
4.  The PXE Client responds to the DHCP Offer with a DHCP Request, where
    it officially requests the IP configuration information from the
    regular DHCP server.
5.  The regular DHCP server responds back with an ACK (acknowledgement),
    letting the client know it can use the IP configuration information
    it requested.
6.  The client now has its IP configuration information, TFTP Server
    name, and boot file name and it initiate a TFTP transaction to
    download the boot file.

## Environment

Tested working with:

  OS Version                         FOG Version
  ---------------------------------- -------------------------------------------
  Ubuntu 10.04 LTS x64               Fog 0.29
  Ubuntu 10.04 LTS x32,x64           Fog 0.32, Fog 1.0.1, Fog 1.1.0
  Ubuntu 11.04 x32, x64              Fog 0.32, Fog 1.0.1, Fog 1.1.0
  Ubuntu 12.04, 12.10 LTS x32, x64   Fog 0.32, Fog 1.0.1, Fog 1.1.0, Fog 1.2.0
  (k)Ubuntu 13.04, 13.10 x32, x64    Fog 0.32, Fog 1.0.1, Fog 1.1.0, Fog 1.2.0
  (k)Ubuntu 14.04, 14.10 x32, x64    Fog 1.1.0, Fog 1.2.0

-   dnsmasq
-   LTSP Server, further documentation at [Ubuntu
    LTSP/ProxyDHCP](https://help.ubuntu.com/community/UbuntuLTSP/ProxyDHCP).

## Setup and Configuration

1.  First get your desired linux flavor installed
2.  Install FOG (use instructions on wiki user manual)
3.  Make sure you do a normal server installation, don\'t setup a DHCP
    router address or a DNS server address, also don\'t use FOG as a
    DHCP server.
4.  If you set a MySQL password make sure you change it in
    /var/www/fog/commons/config.php and also in
    /opt/fog/service/etc/config.php
5.  Edit /etc/exports to look like this:
        /images                        *(ro,async,no_wdelay,insecure_locks,no_root_squash,insecure)
        /images/dev                    *(rw,async,no_wdelay,no_root_squash,insecure)
6.  Install dnsmasq using:
        sudo apt-get install dnsmasq
7.  Create /etc/dnsmasq.d/ltsp.conf using the following settings, modify
    as needed:
        # Sample configuration for dnsmasq to function as a proxyDHCP server,
        # enabling LTSP clients to boot when an external, unmodifiable DHCP
        # server is present.
        # The main dnsmasq configuration is in /etc/dnsmasq.conf;
        # the contents of this script are added to the main configuration.
        # You may modify the file to suit your needs.

        # Don't function as a DNS server:
        port=0

        # Log lots of extra information about DHCP transactions.
        log-dhcp

        # Dnsmasq can also function as a TFTP server. You may uninstall
        # tftpd-hpa if you like, and uncomment the next line:
        # enable-tftp

        # Set the root directory for files available via FTP.
        tftp-root=/tftpboot

        # The boot filename.
        dhcp-boot=pxelinux.0

        # rootpath option, for NFS
        dhcp-option=17,/images

        # kill multicast
        dhcp-option=vendor:PXEClient,6,2b

        # Disable re-use of the DHCP servername and filename fields as extra
        # option space. That's to avoid confusing some old or broken DHCP clients.
        dhcp-no-override

        # PXE menu.  The first part is the text displayed to the user.  The second is the timeout, in seconds.
        pxe-prompt="Press F8 for boot menu", 3

        # The known types are x86PC, PC98, IA64_EFI, Alpha, Arc_x86,
        # Intel_Lean_Client, IA32_EFI, BC_EFI, Xscale_EFI and X86-64_EFI
        # This option is first and will be the default if there is no input from the user.
        pxe-service=X86PC, "Boot from network", pxelinux

        # A boot service type of 0 is special, and will abort the
        # net boot procedure and continue booting from local media.
        pxe-service=X86PC, "Boot from local hard disk", 0

        # If an integer boot service type, rather than a basename is given, then the
        # PXE client will search for a suitable boot service for that type on the
        # network. This search may be done by multicast or broadcast, or direct to a
        # server if its IP address is provided.
        # pxe-service=x86PC, "Install windows from RIS server", 1

        # This range(s) is for the public interface, where dnsmasq functions
        # as a proxy DHCP server providing boot information but no IP leases.
        # Any ip in the subnet will do, so you may just put your server NIC ip here.
        # Since dnsmasq is not providing true DHCP services, you do not want it
        # handing out IP addresses.  Just put your servers IP address for the interface
        # that is connected to the network on which the FOG clients exist.
        # If this setting is incorrect, the dnsmasq may not start, rendering
        # your proxyDHCP ineffective.
        dhcp-range=192.168.1.10,proxy

        # This range(s) is for the private network on 2-NIC servers,
        # where dnsmasq functions as a normal DHCP server, providing IP leases.
        # dhcp-range=192.168.0.20,192.168.0.250,8h

        # For static client IPs, and only for the private subnets,
        # you may put entries like this:
        # dhcp-host=00:20:e0:3b:13:af,10.160.31.111,client111,infinite
8.  Restart dnsmasq with
        sudo service dnsmasq restart

**Note:** After getting everything working, you can change the timeout
to 0 on the line:

    pxe-prompt="Press F8 for boot menu", 3

## DNSMASQ settings for iPXE

This information pertains to FOG 0.33 and the new iPXE boot method.

In order to continue to use dnsmasq to dole out ip addresses and to help
find the boot file, some changes need to be made to force the boot file
to load the iPXE boot file.

**\*\*\*FIRST\*\*\* Update the schema by navigating to your fog
management page and install the update.**

Make the following changes to your ltsp.conf file

    # Don't function as a DNS server:
    port=0

    # Log lots of extra information about DHCP transactions.
    log-dhcp

    # Dnsmasq can also function as a TFTP server. You may uninstall
    # tftpd-hpa if you like, and uncomment the next line:
    # enable-tftp

    # Set the root directory for files available via FTP.
    tftp-root=/tftpboot

    # The boot filename, Server name, Server Ip Address
    dhcp-boot=undionly.kpxe,,x.x.x.x

    # rootpath option, for NFS
    #dhcp-option=17,/images

    # kill multicast
    #dhcp-option=vendor:PXEClient,6,2b

    # Disable re-use of the DHCP servername and filename fields as extra
    # option space. That's to avoid confusing some old or broken DHCP clients.
    dhcp-no-override

    # PXE menu.  The first part is the text displayed to the user.  The second is the timeout, in seconds.
    pxe-prompt="Press F8 for boot menu", 3

    # The known types are x86PC, PC98, IA64_EFI, Alpha, Arc_x86,
    # Intel_Lean_Client, IA32_EFI, BC_EFI, Xscale_EFI and X86-64_EFI
    # This option is first and will be the default if there is no input from the user.
    pxe-service=X86PC, "Boot from network", undionly

    # A boot service type of 0 is special, and will abort the
    # net boot procedure and continue booting from local media.
    #pxe-service=X86PC, "Boot from local hard disk", 0

    # If an integer boot service type, rather than a basename is given, then the
    # PXE client will search for a suitable boot service for that type on the
    # network. This search may be done by multicast or broadcast, or direct to a
    # server if its IP address is provided.
    # pxe-service=x86PC, "Install windows from RIS server", 1

    # This range(s) is for the public interface, where dnsmasq functions
    # as a proxy DHCP server providing boot information but no IP leases.
    # Any ip in the subnet will do, so you may just put your server NIC ip here.
    # Since dnsmasq is not providing true DHCP services, you do not want it
    # handing out IP addresses.  Just put your servers IP address for the interface
    # that is connected to the network on which the FOG clients exist.
    # If this setting is incorrect, the dnsmasq may not start, rendering
    # your proxyDHCP ineffective.
    dhcp-range=10.0.0.10,proxy

    # This range(s) is for the private network on 2-NIC servers,
    # where dnsmasq functions as a normal DHCP server, providing IP leases.
    # dhcp-range=192.168.0.20,192.168.0.250,8h

    # For static client IPs, and only for the private subnets,
    # you may put entries like this:
    # dhcp-host=00:20:e0:3b:13:af,10.160.31.111,client111,infinite

Save your file and restart your dnsmasq service with the following
command:

    sudo service dnsmasq restart

Make a symlink for the undionly.kpxe file so dnsmasq can find it.

    cd /tftpboot
    sudo ln -s undionly.kpxe undionly.0

OR

    cd /tftpboot
    cp undionly.kpxe undionly.0

## Additional Steps for 12.04.4, 12.04.5, 14.04, 14.10

In Specific, when starting DNSMASQ you receive the following error:

    dnsmasq: failed to create listening socket for port 53: Address already in use failed!

If you are using Ubuntu version 12.04.4, 12.04.5, 14.04, 14.10,
dnsmasq-base is already installed on your system and in use by the
network-manager.

Attempting to start the dnsmasq service after installation will lead to
an error, the error mentioned above. To fix this error:

1.  Open terminal and issue the following command:
        sudo nano /etc/NetworkManager/NetworkManager.conf
2.  Remove the line
        dns=dnsmasq
3.  Now we need to restart the network service
        sudo service network-manager restart
4.  This should resolve issues with getting dnsmasq to start.
5.  Issue the following command:
        sudo service dnsmasq restart

## Serving ProxyDHCP to multiple subnets

If you are serving ProxyDHCP to multiple subnets some changes must be
made to your switches/routers and your server config.

1.  Modify your /etc/dnsmasq.d/ltsp.conf file by adding the subnet mask
    option to line:
        dhcp-range=192.168.1.10,proxy

    to make it

        dhcp-range=192.168.1.10,proxy,255.255.0.0

    which will serve all 192.168.x.x subnets. If you are using 10.x.x.x
    addressing, use subnet mask \"255.0.0.0\" (8-bit) and if you are
    using 172.16.x.x, use subnet mask \"255.240.0.0\" (12 bit).
    Basically set the subnet mask so that all subnets on which ProxyDHCP
    should answer are covered.

    If you don\'t do this, the ProxyDHCP server will not respond to DHCP
    requests for hosts outside of it\'s own subnet.
2.  Add an IP Helper/DHCP Relay record to your router or switch so the
    DHCP broadcasts are sent to your normal DHCP server AND the Fog
    server.

## References

I gathered a lot of my ideas from peoples\' questions on the FOG forums
and the Ubuntu documentation on the [LTSP
proxyDHCP](https://help.ubuntu.com/community/UbuntuLTSP/ProxyDHCP)
server, so thanks to them. Junkhacker - for help with iPXE chainloading
jbsclm - for his work on figuring out how to chainload 0.33b with 0.32
pxelinux.0 <http://forum.ipxe.org/showthread.php?tid=6077> -
documentation on chainloading with dnsmasq

## Troubleshooting

As ProxyDHCP intercepts DHCP requests, it starts its own internal
checks. If it can\'t find the boot-file that is supposed to be assigned,
it tells the requesting system there is nothing to find.

If it finds the file, it will send out the info as normal.

```{=mediawiki}
{{:TCPDump}}
```
Using the above method and filter, this is what a **BROKEN** dnsmasq
(ProxyDHCP) conversation looks like:

![[Broken_dnsmasq.png]]

In this case, dnsmasq boot file name is not configured correctly, the
boot file does not exist, or TFTP is not configured properly.

## Additional Info

A ProxyDHCP server can also help deal with PXE Clients that do not work
with seperate DHCP and TFTP servers using option 66 &67 (Windows), or
next-server and filename (Linux). This can resolve situations where the
clients are getting the tftp server IP address and filename, but are
having issues with the TFTP Transaction, such as: PXE-T01: File not
found, and other errors.

This has successfully resolved issues with:

  Device                  NIC
  ----------------------- --------------------------------------------
  Acer Iconia Tab w500p   Asix AX88772B USB to Fast Ethernet adapter
  Compal JHL91            Realtek RTL8139


