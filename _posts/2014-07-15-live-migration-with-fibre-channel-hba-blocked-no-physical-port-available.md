---
title:  "Live Migration with Fibre Channel HBA Blocked: No Physical Port Available"
date:   2014-07-15 12:00:00
categories: ["Hyper-V","Windows Server","Virtual Fibre Channel"]
tags: ["Hyper-V","Windows Server","Virtaul Fibre Channel"]
---
Was confronted today with a Hyper-V host which did not accept inbound Live Migrations of VMs equipped with virtual fibre channel.

![](/images/2014-07/npiverror.jpg)

At first I thought about the vPort issue I blogged about earlier [here](http://bgelens.nl/live-migration-blocked-by-npiv-ports-still-in-use-solved/). This was however not the case (not only because the error states no PHYSICAL ports are available instead of VIRTUAL ports, I should read error messages before I start troubleshooting them!). My function described in the earlier blog post didn’t show any results (which is good :) ) but for one host an error was raised: ```Get-CimInstance : The WS-Management service cannot process the request. The class MSFC_FibrePortNPIVAttributes does not exist in the root/wmi namespace```.

I triple checked: ```gwmi -Namespace root\wmi -List | ?{$_.name -like “MSFC*”}``` the class was indeed missing… what happened?

Something (or somebody) had decided to rebuild the WMI repository. There are actually a lot of dangerous blogs written about this wonder method to fix e.g. VMM connectivity issues. Better think twice is my advice before you do something like this. Rebuilding the WMI repository can have some nasty side effects like missing classes :( To fix this issue run the following command: ```mofcomp -N: “C:\Program Files\Microsoft System Center 2012 R2\Virtual Machine Manager\setup\NPIV.mof”``` A warning is actually given that the MOF file is missing information for a potential future rebuild and because of this, the content won’t survive the rebuild!

Reboot the host and you are back in business!

HTH Ben