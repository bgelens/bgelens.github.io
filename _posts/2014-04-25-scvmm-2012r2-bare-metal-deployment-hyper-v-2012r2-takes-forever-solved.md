---
title:  "SCVMM 2012R2 Bare Metal Deployment Hyper-V 2012R2 takes forever SOLVED!"
date:   2014-04-25 12:00:00
categories: ["Hyper-V","SCVMM","Windows Server"]
tags: ["Hyper-V","PowerShell","SCVMM","Bare Metal Deployment","Windows Server"]
---
One thing I noticed post upgrading a Private Cloud to the 2012R2 wave was that Bare Metal Deployment was seriously impacted duration wise.
“Getting Ready” took over half an hour!!!

![](/images/2014-04/getting-ready.jpg)

This makes the deployment half an hour longer than with Server 2012(R1).
I decided to investigate the issue and report my findings here.

Spoiler alert, I got it fixed as well :-)

So I use HP BL460c Gen8 for my deployments and configure them with a SCVMM physical Computer profile which assigns a static IP to the management NIC. Herein lays the problem but I will explain that a bit later on. The static IP is assigned during the “specialize phase” which is run just after the Hardware Installation step. Then the domain join process takes place and it just can’t resolve with DNS. Eventually (after half an hour) it will succeed but this process should work instantaneously.

When investigating the setupact.log file you can see a lot of “[DJOIN.EXE] Unattended Join: DsGetDcName failed: 0x54b, last error is 0x0, will retry in 10 seconds...” warnings. You can view this file while waiting for “Getting ready” by pressing shift + F10 which will open a cmd.exe box or view it post install. The log can be found c:\Windows\Panther\UnattendGC.

While waiting in this screen of “Getting ready” I opened a command box by pressing Shift + F10 and started doing some basic testing:
<ul>
	<li>Pinged my VMM server by IP address (successful)</li>
	<li>Pinged my VMM server by FQDN (failed, host not found)</li>
	<li>Pinged my Active Directory name (failed, host not found)</li>
	<li>Pinged my DC by FQDN (failed, host not found)</li>
	<li>Nltest /dsgetdc:DNSDomainName (failed, no such domain)</li>
	<li>Nltest /dsgetdc:NETBIOSDomainName (success)</li>
</ul>
As you can see, all issues where DNS related. I then decided to try something rigorous and stopped the dnscache service. When I did this, the host rebooted immediately. I figured because something went terribly wrong. When the host came back up VMM finished the Bare Metal deployment and after some investigation I found that as soon as I killed the dnscache service the djoin succeeded and the process was successfully finished.

So what is the deal? During deployment DHCP is used until the specialize phase sets the IP address on the NIC. All the time DNS works just fine until just after this crucial moment. I guess this is just small bug in the windows setup process when using generalized images. I’m thinking of posting this finding on Connect but I can’t seem to find an opening for Windows bugs….

So you said you solved it right? Yes I did, well sort of. I made sure the dnscache service is killed during the specialize phase by adding a synchronous command in the unattended file I use for my Hyper-V deployments.

![](/images/2014-04/unattend1.jpg)

Make sure to make the order number 1 as SCVMM will merge its own generated unattended file with the manually provided one. When set to a number 2 or 3 it will conflict with the mpclaim commands added by SCVMM and will result in a failed deployment.

![](/images/2014-04/computerprofile.jpg)

That’s it for today! I have some more SCVMM with Hyper-V 2012R2 deployment stuff coming so stay tuned!