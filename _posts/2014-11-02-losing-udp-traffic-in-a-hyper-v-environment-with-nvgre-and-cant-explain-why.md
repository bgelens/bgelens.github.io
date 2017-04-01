---
title:  "Losing UDP traffic in a Hyper-V environment with NVGRE and can't explain why?"
date:   2014-11-02 12:00:00
categories: ["Hyper-V","Windows Server"]
tags: ["Hyper-V","Windows Server","SDN","NVGRE"]
---
I was working on a project where I had to setup Hybrid connectivity. NVGRE isolated tenant network connected up to an Azure vNET using IPSEC VPN over a HNV gateway. Awesome stuff :-)

After the IPSEC connection was established I deployed a VM in Azure and tested the connection. All looked fine until I tried to domain join the Azure VM to the tenant AD.

I started basic troubleshooting using DNS lookups. Resolve-DnsName resulted in error but Resolve-DnsName -TcpOnly came through perfectly. Other unexplained connectivity issues between tenant VMs which were spread over different Hyper-V hosts where experienced by other project members as well.

As I didn't setup the fabric, I asked which physical network cards were being used. I was informed that a mix of Broadcom 10G cards where installed. I remembered that Broadcom cards have the <strong>encapsulating task offload</strong> setting enabled by default and this had caused me issues in the past with another costumer. After disabling this setting on the Hyper-V hosts used with the HNV gateway and the one hosting the tenant DC, DNS lookups where functioning properly. I joined the VM in Azure and verified solid connectivity.

The customer then disabled this setting on all Broadcom NICs on every Hyper-V hosts. The lesson learned here is to test, test, test and do not rely on the default settings of a NIC vendor.

HTH, Ben
