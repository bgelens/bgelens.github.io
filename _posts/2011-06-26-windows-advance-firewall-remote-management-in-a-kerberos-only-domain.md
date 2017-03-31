---
title:  "Windows Advanced Firewall Remote Management in a Kerberos only Domain"
date:   2011-06-26 12:00:00
categories: ["Windows Server","Active Directory"]
tags: ["Kerberos","Windows Server","Windows Advanced Firewall","Windows Firewall","NTLM"]
---
I know not a lot of people created or transitioned to a forest/domain which expelled NTLM in total, but I know for sure when time will progress a lot more people will take this step.
When the time comes and you’ll take the small step for men, but giant step for your authentication security you’ll be confronted with some nasty bugs of which I will highlight one in this post.

In this scenario:
You need to remotely manage Windows Advanced Firewall in your Kerberos only environment. You’ll start up mmc and add the Windows Advanced Firewall snap-in. You’ll connect it with a remote computer and….. bummer. Authentication fails. Re-allow NTLM and… authentication is successfull! By now you know windows advanced firewall remote management falls back to NTLM
authentication. If you want to know why, you’ll need to capture the traffic.

![](/images/2011-06/sname1.jpg)

Through the packet capture you’ll find that in the KRB_AP-REQ the sname is not in the expected “HOST/FQDN” or “HOST/Netbios” but “HOST/NULL/FQDN.” or “HOST/NULL/Netbios.” (translates into “HOST//FQDN.” or “HOST//Netbios.”).
Keep special notice to the trailing DOTS. So we know now our MMC instance is the originator of this authentication problem. (I’ve opened a connect bug call a little while ago to report this with Microsoft.) We also know that we can’t manipulate this behavior (only Microsoft Devs are able to) so we work around this problem by setting *wrong* SPN’s to compensate!
We will run ```SetSPN -A HOST//FQDN. computername``` and ```SetSPN -A HOST//Netbios. computername```. Where computername, FQDN and Netbios is the computer from which we want to remotely manage Windows Advanced Firewall (Run this with Domain Admin privileges).
By doing this, the TGS is able to find the security principle we want to manage and thus is able to send of the necessary tickets!
Hope you’ll find this post useful!