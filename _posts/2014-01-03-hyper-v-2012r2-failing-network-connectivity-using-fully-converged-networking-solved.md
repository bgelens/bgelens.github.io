---
title:  "Hyper-V 2012R2 failing network connectivity using fully converged networking SOLVED!"
date:   2014-01-03 12:00:00
categories: ["Hyper-V"]
tags: ["Hyper-V","VMQ","SCVMM"]
---
<strong>Update: </strong>The real fix is not to disable VMQ on the Management vNIC but rather fix the MAC address issue itself. I did that as well, you can find it <a href="http://wp.me/p1COfr-4w" target="_blank">here</a>!
At least in my case that is :)

On a project I’m currently working on I am using SCVMM 2012R2 to bare-metal deploy Hyper-V 2012R2 on HP DL380P gen8 / BL460C gen 8 system.

The DL380P are equipped with 530FLR 2port 10GBe (Broadcom BCM57810) and the BL460C are equipped with 554FLB and 554M (both Emulex) adapters. I won’t go into too much details about the setup but basically it comes down to deploying a logical switch on a team and attaching the management, live migration and cluster network adapters to it.

While testing this setup out I was confronted by erratic network connectivity issues. Cluster validation wasn’t able to connect via WMI (so effectively, no cluster could be build), RDP would drop sporadically, etc. I found some excellent blogs about these issues on hyper-v.nu I thought my issues where related to. For more details see: <a href="http://www.hyper-v.nu/archives/hvredevoort/2013/12/november-2013-ur-kb2887595-causing-network-problems/" target="_blank">http://www.hyper-v.nu/archives/hvredevoort/2013/12/november-2013-ur-kb2887595-causing-network-problems/</a>.

In my case I also went through a process of elimination as described in these blogs. Eventually I wound up disabling VMQ on the teamed network adapters and everything worked fine, except of course this isn’t a viable solution (you might as well rip those 500 dollar NICs out and replace them with Realtec’s right ;) ).

So I wasn’t happy about it and theorized for a bit after reading this excellent series of posts <a href="http://blogs.technet.com/b/networking/archive/2013/09/10/vmq-deep-dive-1-of-3.aspx" target="_blank">http://blogs.technet.com/b/networking/archive/2013/09/10/vmq-deep-dive-1-of-3.aspx</a> (2 and 3 can be found here as well). Long story short, using VMQ will create queues based on the MAC address (imagine it to be a sort of a routing table). When you deploy a Hyper-V host without dedicated management adapters you will end up with 3 (v)NICs that have the same MAC address, the multiplexer NIC will inherit the MAC from the physical NIC, the virtual NIC which is created afterwards will inherit the MAC from the multiplexer NIC. By default the port profile for host management will have VMQ enabled.

Looking at the queues post deployment you will see something like this:

![](/images/2014-01/vmqbefore.jpg)

As you can see, there are 2 queues mapped to the same MAC address.

After disabling VMQ on the port profile:

![](/images/2014-01/portprofile.jpg)

And remediating the host you will end up with:

![](/images/2014-01/remediate.jpg)

Voila! No more “routing” issues.

![](/images/2014-01/vmqafter.jpg)

Since there is no need for VMQ on the management adapter, this is a viable solutions.

Hope this helps!