---
title: 'VPN Tuneel to Checkpoint Gateway with Openswan - Intro'
date: 2012-09-23
permalink: /posts/2012/09/openswan-intro/
tags:
  - checkpoint
  - ipsec
  - openswan
  - vpn tunnel
---

### Overview

Recently I had the situation where I had to connect to a VPN via a Checkpoint Security Gateway (R70.x) via Linux with IPSec stack. While there were a lot of posts in the net regarding Openswan to which I am very grateful, I still had to tweak configurations from here and there until I could get it working with collaboration from the admin at the gateway side as well. There is also this great book called 'Building and Integrating Virtual Private Networks with Openswan' by Ken Bantoft and Paul Wouters which is like the recommended text for Openswan stuff. For Openswan, this is the background you need for get it working.

### Installing Openswan

To initially install openswan, you can get it from the package manager of your distro, compile it from source, download a binary or whatever your preference. Mainly this post follows the Debian/Ubuntu setup even though I did try it from Fedora 15 and got the initial tunnel established as well. I think with a few tweaks here and there you can get it to work for any RPM based setup as well without any issue.





For Openswan, you can use pre-shared keys or X.509 certificates. Our setup was with X.509 certificates. For a Ubuntu, you can just use

`sudo apt-get install openswan`

It would ask for certificate details at the time of installation where you would need to have prepared-before hand the following documents in PEM format. If you are using untrusted certificates, it would be best to ask your Checkpoint Gateway admin to generate your certificate with the Checkpoint Gateway authenticating as the CA. You might get a .p12 file from which you would have to extract these documents separately. There are many sites that tell you how to convert a PKCS#12 format to various other formats to suit your needs... :-)

1. Client side certificate
2. Client side private key
3. CA certificate (Use the CA certificate of the Gateway for this when working with untrusted [not signed by a trusted CA like Verisign] certificates)

Alternatively, you can run '**dpkg-reconfigure openswan**' at anytime to get that certificate specifying step. This is much easier than manually copying the certificates to proper folders (e.g. CA cert to **/etc/ipsec.d/cacert**, keys to **/etc/ipsec.d/private** etc.) and configuring.

After this you have to configure some initial parameters so that ipsec  works properly.

### Initial Setup

Before the initial setup, you might have to start Openswan by '**service ipsec start**' and run '**ipsec verify**'. Oh, and please drop into a root shell if you don't want to type 'sudo' or 'su -c' or whatever in front of every command. The output of ipsec verify might be similar to this.

```
Checking your system to see if IPsec got installed and started correctly:
Version check and ipsec on-path                              [OK]
Linux Openswan U2.6.37/K2.6.xx.x-1.xx.xx (netkey)
Checking for IPsec support in kernel                         [OK]
 SAref kernel support                                        [N/A]
 NETKEY:  Testing XFRM related proc values                   [FAILED]

  Please disable /proc/sys/net/ipv4/conf/*/send_redirects
  or NETKEY will cause the sending of bogus ICMP redirects!

 [FAILED]

  Please disable /proc/sys/net/ipv4/conf/*/accept_redirects
  or NETKEY will accept bogus ICMP redirects!

 [OK]
Checking that pluto is running                               [OK]
 Pluto listening for IKE on udp 500                          [OK]
 Pluto listening for NAT-T on udp 4500                       [OK]
Two or more interfaces found, checking IP forwarding         [OK]
Checking NAT and MASQUERADEing                               [OK]
Checking for 'ip' command                                    [OK]
Checking /bin/sh is not /bin/dash                            [OK]
Checking for 'iptables' command                              [OK]
Opportunistic Encryption Support                             [DISABLED]
```

You might actually see a 'FAILED' for 'Checking NAT and MASQUERADEing' as well if ip forwarding is not enabled. In the **/etc/sysctl.conf** you would have to uncomment or add the following lines. [Caution: Do not change other existing lines]

```
net.ipv4.ip_forward = 1

net.ipv4.conf.default.rp_filter = 0

# Disable ICMP redirects for OpenSwan NETKEY stack and other required options


# Below two lines don't seem to work when enabled :-(

#net.ipv4.conf.all.accept_redirects = 0

#net.ipv4.conf.all.send_redirects = 0


net.ipv4.icmp_ignore_bogus_error_responses = 1

net.ipv4.conf.all.log_martians = 0
```

Now, use the 'sysctl -p' command to apply these settings. After that run 'ipsec verify' again. If you are still seeing the 'FAILED' checks even after that, use these commands to manually disable ICMP redirects.


`for f in /proc/sys/net/ipv4/conf/*/send_redirects; do echo 0 &gt; $f; done`

`for f in /proc/sys/net/ipv4/conf/*/accept_redirects; do echo 0 &gt; $f; done`

Well, that should fix your network configurations for Openswan/IPSec. Do not worry about Opportunistic Encryption being disabled or SAref kernel support not being available. Other parameters should be in [OK] status for the default installation configuration of Openswan. Now we move on to some info on the terms for interested readers. Those who know those stuff or would like to avoid those details can skip the next section.

### Terminology

Now for some overview of the terms and technologies related to setting up a VPN connection via Openswan. First of all, I need to tell you that I do not have in-depth knowledge about any of these technologies,protocols mentioned here. But it helps to be familiar with some of the terms.

#### IPSec

Of course, IPSec is an established standard and Openswan uses IPSec to establish the tunnel. You can learn about IPSec from any standard course on Networking. When connecting with IPSec, there are several phases that the connection goes through when establishing a tunnel. You start from Initiating IKE (Internet Key Exhcange) and go to IKE Phase One, IKE Phase Two and finally after IPSec SAs are established, you can start transferring data. You can see the status of different phases in the output of Openswan or in the logs for Openswan when establishing a tunnel. This information can help you a lot in troubleshooting your connection. If you want to know about IKE and IPsec, here is a nice link. Refer to the contents section of the article and it allows you to jump to the specifics of all the related stuff.

#### Left and Right

Openswan refers to the two endpoints of the tunnel as left and right and automatically identifies which peer it is running on based on the configuration. This left and right setup is so that you can copy the same configuration to both endpoints with minimal modification. Please feel free to setup your client configuration as left or right according to your preference. :-)

#### Protocol Stack

For Openswan running on Linux, you have to specify a protocol stack. The old stack was KLIPS where you had to manually specify forwarding rules and had more configuration. The newer stack is Netkey and now most kernels do not support KLIPS by default. If you want to go the KLIPS way, here are some posts about compiling KLIPS into your kernel.

* [Openswan Wiki Post](https://www.openswan.org/projects/openswan/wiki/Building_and_Installing_an_SAref_capable_KLIPS_version_for_DebianUbuntu)
* [Ubuntu Forum Post](http://ubuntuforums.org/showthread.php?t=1721842)

You might have some trouble building KLIPS with NAT-T support. One important difference between KLIPS and Netkey is that KLIPS creates a separate virtual interface for the tunnel established. This might be much more easier to configure in certain cases.

Netkey is still evolving and so not an old and proven technology like KLIPS. It automatically does all the necessary configurations for routing and establishing a tunnel.

#### NAT-T

Nat traversal is needed because most of the time we are probably sitting behind some Nat. So it is always advisable to enable NAT-Traversal or NAT-T for Openswan. This needs to be enabled at both ends. 

There are loads of other stuff related to configuring Openswan that needs to be mentioned. I will mention the detailed configuration stuff in [another post](http://randomthoughtsofbloggerrandom.blogspot.com/2012/10/connecting-to-checkpoint-gateway-with.html). Anyway, the sources mentioned here are pretty good and complete, so you can use them to go ahead and configure stuff on your own if you'd like to hack a little bit and get things working.

### Sources

[http://www.jacco2.dds.nl/networking/linux-l2tp.html](http://www.jacco2.dds.nl/networking/linux-l2tp.html)
[https://www.openswan.org/projects/openswan/wiki/Connecting_to_the_CheckPoint_VPN-1_NG65_firewall](https://www.openswan.org/projects/openswan/wiki/Connecting_to_the_CheckPoint_VPN-1_NG65_firewall)
