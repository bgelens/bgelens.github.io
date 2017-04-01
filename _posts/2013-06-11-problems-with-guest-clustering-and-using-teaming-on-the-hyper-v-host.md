---
title:  "Problems with Guest Clustering and using Teaming on the Hyper-V host?"
date:   2013-06-11 12:00:00
categories: ["Hyper-V"]
tags: ["Hyper-V"]
---
Check out others having the same issue and my workaround at:</br>
[http://social.technet.microsoft.com/Forums/en-US/winserverhyperv/thread/2a83829b-5d85-4a00-90a7-2e18ce12df47](http://social.technet.microsoft.com/Forums/en-US/winserverhyperv/thread/2a83829b-5d85-4a00-90a7-2e18ce12df47)

>I experienced the guest cluster failure when they are spread over 2 Hyper-V 2012 nodes as well. In my case we use LACP trunks with Transport ports address hashing on the host team. When assigning the Hyper-V ports algorithm there was still no joy.</br></br>
>I’ve figured out a workaround though (Off course this is not THE solution because extra complexity is added)
>In Windows Firewall I created a custom connection rule for port UDP 3354 to require inbound and outbound authentication with the computers Kerberos ticket. It will setup an ESP tunnel which encapsulates the Cluster (Heartbeat) Ping. This will make sure the guest cluster reports and acts healthy. [http://support.microsoft.com/kb/2455128](http://support.microsoft.com/kb/2455128)</br></br>
>I’ve created a GPO for the cluster nodes and under it’s computer configuration -> Windows Settings -> Security Settings -> Windows Firewall with Advanced Security I’ve created a new Custom Connection Security Rule.</br></br>
>For Endpoint 1 and 2 I’ve configured the entire heartbeat subnet and both cluster nodes public IP addresses (alternate path). So for example in Endpoint 1 and 2 you enter 192.168.1.0/24 (cluster subnet), 10.10.10.1 (node 1) and 10.10.10.2 (node 2).</br></br>
>Authentication should be configured to “Require for in and outbound” with authentication method Computer (Kerberos V5).</br></br>
>Protocols should be configured for type UDP. At endpoint 1 you configure All ports and at endpoint 2 specific 3343.</br></br>
>Select all profiles, name it, gpupdate and your done.

## Update01: 
Seems the problem was caused by checksum offloading. I've verified the guest clusters to be still operational when IPsec is removed.</br>[http://social.technet.microsoft.com/Forums/en-US/winserverClustering/thread/51557f60-f2e8-47fd-a1a3-8697bd75513f](http://social.technet.microsoft.com/Forums/en-US/winserverClustering/thread/51557f60-f2e8-47fd-a1a3-8697bd75513f)