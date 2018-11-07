---
title: 'Connecting to Checkpoint Gateway with Openswan - The details'
date: 2012-10-23
permalink: /posts/2012/10/connecting-to-checkpoint/
tags:
  - checkpoint
  - openswan
  - ipsec
  - vpn tunnel
---

Overview
In a previous post I explained the basic pre-requisites that you have to setup before you try to establish a VPN tunnel with Openswan. Also, I tried to explain some related terminology to the best of my understanding (That 'to the best of my understanding' part is important ... :-) ). In this post, the detailed configuration that worked for me is given.



Check-list


Before proceeding further, it is always prudent to check that the following steps have been performed.

Proper certificates for CA and the client machine and the private key for the client has been placed in the proper folders (i.e. /etc/ipsec.d/cacerts,/etc/ipsec.d/certs,/etc/ipsec.d/private)
IKE uses port 500 by default and NAT-T uses port 4500. It is better to keep both these ports open for UDP traffic by your firewall.
Try logging through a Windows machine to ensure that the Checkpoint Gateway is up and a VPN tunnel can be established.
Run 'ipsec verify' to determine that all configurations are OK (No 'FAILED' rows)
Run 'ip xfrm state' to verify that no previously established tunnels are still up.


Details


A Very Important Detail


Something that had me stuck for weeks was that Openswan indicated that a tunnel was established, but no traffic could pass through. The reason is simply that for the default configuration of checkpoint with the client provided for windows by Checkpoint, a tunnel within a tunnel is established. This internal tunnel has an ip range of 10.254.1.0/24 and all the firewall rules at the gateway are written based on that IP range. When Openswan connects to the Checkpoint gateway, the source IP of the connecting machine is taken as the client end IP for the tunnel. So you will have to contact the system administrator at the Checkpoint Gateway and ask him/her to adjust the firewall rules accordingly so that traffic passes through for the required traffic.



Detailed Configuration


Ok, here is the configuration that worked for me. The top level configuration for openswan which is usually at /etc/ipsec.conf looked like this.



# basic configuration
config setup
        # Do not set debug options to debug configuration issues!
        # plutodebug / klipsdebug = "all", "none" or a combination from below:
        # "raw crypt parsing emitting control klips pfkey natt x509 dpd private"
        # eg:
        plutodebug="control parsing"
        #
        # enable to get logs per-peer
        # plutoopts="--perpeerlog"
        #
        # Again: only enable plutodebug or klipsdebug when asked by a developer
        #
        # NAT-TRAVERSAL support, see README.NAT-Traversal
        nat_traversal=yes
        # exclude networks used on server side by adding %v4:!a.b.c.0/24
        virtual_private=%v4:10.0.0.0/8,%v4:192.168.0.0/16,%v4:172.16.0.0/12,%v4:!192.x.x.0/24
        # OE is now off by default. Uncomment and change to on, to enable.
        oe=off
        # which IPsec stack to use. netkey,klips,mast,auto or none
        protostack=netkey



Also you might have to edit /etc/ipsec.secrets or to the file that is pointed to by it and add a line similar to following to tell Openswan where your private key is located.

: RSA /etc/ipsec.d/private/client-key-out.pem


The connection specific configuration looks like this.



conn companyvpn
        # Left side is Openswan
        left=%defaultroute
        leftid=%fromcert
        leftsubnet=10.x.x.x/32
        leftrsasigkey=%cert
        leftcert=checkpoint-clcert.pem

        # Right side is Checkpoint
        right=x.x.x.x    # The public IP of the checkpoint gateway should come here
        rightsubnet=192.x.x.x/32
        rightcert=checkpoint-cacert.pem

        # config
        type=tunnel
        keyingtries=3
        rekey=no
#       forceencaps=yes
#       disablearrivalcheck=no
        authby=rsasig

# Please contact the administrator of the gateway to get the exact details for below settings
        # Modify settings if "NO_PROPOSAL_CHOSEN" comes.
        pfs=no
#       aggrmode=yes
        ike=3des-sha1;modp1536
#       keyexchange=ike
#       keylife=1h
        auto=add


Now you would need to restart ipsec service to make the configuration changes effective.

sudo /etc/init.d/ipsec restart

Now to establish the connection, use the following command.

sudo ipsec auto --up companyvpn

If all goes well, you will get a message like follows with a successfully established connection.

104 "companyvpn" #1: STATE_MAIN_I1: initiate
003 "companyvpn" #1: received Vendor ID payload [draft-ietf-ipsec-nat-t-ike-02_n] method set to=106 
106 "companyvpn" #1: STATE_MAIN_I2: sent MI2, expecting MR2
003 "companyvpn" #1: NAT-Traversal: Result using draft-ietf-ipsec-nat-t-ike-02/03: i am NATed
108 "companyvpn" #1: STATE_MAIN_I3: sent MI3, expecting MR3
004 "companyvpn" #1: STATE_MAIN_I4: ISAKMP SA established {auth=OAKLEY_RSA_SIG cipher=oakley_3des_cbc_192 prf=oakley_sha group=modp1536}
117 "companyvpn" #2: STATE_QUICK_I1: initiate
003 "companyvpn" #2: ignoring informational payload, type IPSEC_RESPONDER_LIFETIME msgid=2a306520
004 "companyvpn" #2: STATE_QUICK_I2: sent QI2, IPsec SA established tunnel mode {ESP=&gt;0x74649578 &gt;0x7695245e dpd="none}" natd="none" natoa="none" pre="pre" xfrm="3DES_0-HMAC_SHA1"&gt;
You can try to ping the other end of the tunnel and also try to telnet to some service to check whether the connection was established properly. But chances are that it will not succeed the first time you try, in which case you might have to tweak some settings to get it working.



Also, you can make use of the logs at /var/log/auth.log (The exact file might differ from one distribution to another) to get the error messages that come out of pluto (the keying daemon for IPSec). Also a command like 'tcpdump' (or a tool like wireshark if you have a GUI based distro) can be very helpful in trying to troubleshoot and resolve any problems that you encounter. Good luck hacking into your ipsec configuration!





Sources
http://www.jacco2.dds.nl/networking/linux-l2tp.html
https://www.openswan.org/projects/openswan/wiki/Connecting_to_the_CheckPoint_VPN-1_NG65_firewall
