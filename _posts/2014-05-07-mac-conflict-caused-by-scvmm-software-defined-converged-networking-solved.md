---
title:  "MAC Conflict caused by SCVMM Software Defined / Converged Networking SOLVED!"
date:   2014-05-07 12:00:00
categories: ["Hyper-V","SCVMM","Windows Server"]
tags: ["Hyper-V","PowerShell","SCVMM","SDN","Windows Server","Bare Metal Deployment"]
---
One of the more popular methods these days of configuring Hyper-V host networking involves the use of software defined networking (SDN) because of the flexibility it brings. More and more often I see new systems being bought for Hyper-V purposes with 2 or 4 10GBe ports instead of a multitude of 1GBe ports.

When you have SCVMM in your environment you want to use it to configure SDN / Converged networking on the host so the concept of a logical switch can be used.

I won’t bother in this post explaining all the details of SDN / Converged Networking and how SCVMM deals with logical switches and virtual NICs because I figure when you are reading this, you already know all this ;-).

In an earlier post, <a href="http://bgelens.nl/hyper-v-2012r2-failing-network-connectivity-using-fully-converged-networking-solved/">Hyper-V 2012R2 failing network connectivity using fully converged networking SOLVED!,</a> I blogged about erratic network behavior because of a VMQ issue on the virtual management interface when converged networking is configured through SCVMM. In short, when SCVMM configures a logical switch with a team uplink it will first create the team which inherits the MAC address of the primary LBFO member. It will then create a VMSwitch on top of the team and attaches a vNIC to it for management. The management vNIC on its turn inherits the MAC address of the Multiplexor adapter. This results in a duplicate MAC issue on the host. Disabling VMQ usage on the management vNIC solves the erratic network behavior but does off course not fix the duplicate MAC issue. There is no way in SCVMM 2012R2 to disable the MAC inheritance of the management vNIC which I find a bit strange since Windows will always send out a gratuitous ARP (GARP) message as soon as you configure a new IP address. A GARP message informs the connected switch of the MAC change allowing connectivity to be re-established. There is actually no need for the duplicate MAC address. I’ve spoken with a senior escalation engineer at Microsoft about the SCVMM resulting setup and he informed me he was going to talk to the VMM guys since the setup results in all sorts of problems.

I have developed a fix for the duplicate MAC issue (effectively rendering the previous blog post irrelevant because when done, VMQ can be enabled on the management vNIC again). The fix will allow for a logical switch to exist within SCVMM with a management vNIC attached to it which will be used for SCVMM to host communication without ever needing to walk over to the server (or go through an Out of Band controller) to finalize configuration. You can of course still walk through the process with assistance of a console connection so you can check the results.

I’ll use my preferred setup as the scenario here. You can easily adopt it to your situation. 4 * 10GBe NICs are used to configure two teams. One is used for infra (Management, CSV and Live Migration) network traffic and the other for VM network traffic. The end result is shown in the following illustration.

![](/images/2014-05/final.jpg)

We start with a normal bare metal deployment or a manually configured hosts with 1 of the 4 NICs configured with the management IP address. When bare metal deployment is used, you configure the physical computer profile to assign the management NIC to a physical NIC.

![](/images/2014-05/start.jpg)

So the first thing which needs to be done is to configure a Logical switch on the second designated Infra NIC.

![](/images/2014-05/switchonothernic.jpg)

<em>Remember, you cannot configure a logical switch on the host side because this is purely a SCVMM concept. In my humble opinion there should be a way though.</em>

Then the configuration needs to continue on the host side. You need to capture the IP address either on your notepad or within a PowerShell variable (when PowerShell is used for this configuration remember to install the Hyper-V PowerShell module) from the management NIC and add the management NIC to the team. Then a vNIC should be created which will have the management IP address, DNS servers and DNS settings applied to it. You can accomplish these steps by using the GUI on the host (who has the GUI installed on Hyper-V??) or by running a script on the host either via an SCVMM post GCE using a custom resource which will ensure the script is downloaded to the host and executed locally (this is crucial since connectivity will be temporarily interrupted) or you can run the script on the host console interactively (ILO or locally via a monitor). After the IP address is applied to the vNIC, Windows will send out a GARP message informing the neighboring switch of the MAC change. This will reestablish communication. I’ve added a sample script:

```powershell
Install-WindowsFeature Hyper-V-PowerShell
$ip = Get-NetIPAddress -IPAddress 192.168.1.*
$NICToBeAdded = (Get-NetAdapterHardwareInfo | ? { $_.bus -eq “33” -and $_.function -eq “0” }).Name
Add-NetLbfoTeamMember -Name $NICToBeAdded -Team“infra” -Confirm:$false
start-sleep 5
Add-VMNetworkAdapter -ManagementOS -Name MGMT -SwitchName “infra” -Confirm:$false
$mgmtVnic = Get-NetAdapter -InterfaceDescription “Hyper-V Virtual Ethernet Adapter*”
New-NetIPAddress -InterfaceIndex $mgmtVnic.ifindex -IPAddress $ip.IPAddress -PrefixLength 24 -DefaultGateway 192.168.1.1 -Confirm:$false
Set-DnsClientServerAddress -InterfaceIndex $mgmtVnic.ifIndex -ServerAddresses “192.168.1.2”,“192.168.1.3” -Confirm:$false
Set-DnsClient -InterfaceIndex $mgmtVnic.ifIndex -ConnectionSpecificSuffix lab.local -RegisterThisConnectionsAddress $true -UseSuffixWhenRegistering $true -Confirm:$false
```

A refresh of the host in SCVMM will inform SCVMM of the change. It will take over the configuration as it exists on the host by adding the second adapter to the logical switch and adding a vNIC for management to it (SCVMM is just a database after all :-) ). So, all logical switch functionality stays intact! And no more MAC conflict!

![](/images/2014-05/nomoremacconflict.jpg)

The rest can be added either via the SCVMM GUI or via SCVMM PowerShell scripts. Let me know if you would like me to make a part two of this blog post finalizing the configuration ok? I would like to know it there is any interest before I take the effort :-) .

There are a lot of steps involved with bare-metal deployment of Hyper-V hosts with SCVMM. Don’t just assume it is as simple as told on TechEd because they leave out all the optimizations that need to occur. Please let me know if you are interested in more posts about this subject.