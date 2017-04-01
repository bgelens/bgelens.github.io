---
title:  "Hyper-V Hyper-threading Enabled or Disabled?"
date:   2014-07-20 12:00:00
categories: ["Hyper-V","Windows Server"]
tags: ["Hyper-V","Windows Server"]
---
Ben Armstrong said at TechEd Australia 2013 that from a Hyper-V perspective it doesn’t really make a difference, performance-wise, if you use HT or not for Hyper-V. You can find the session [here](http://channel9.msdn.com/Events/TechEd/Australia/2013/MDC332A){:target="_blank"} and at 35:40 the HT subject begins.

After careful study and testing I’ve come to the conclusion that enabling or disabling HT actually can impact performance and configuration in some cases, especially when high density is NOT the prime objective. 

In this blog I share my notes about the subject hoping it can help you when you are at the design phase.

First, enabling HT will impact the way you size high demand / big scale VMs like SQL servers because the Hyper-V vNUMA calculation algorithm does not take into account that half of the LPs (Logical Processors) are actually HT LPs, this will force you to assign (socket LPs + socket HT LPs +1LP from another socket to cross the NUMA boundary) vCPUs when you want to make use of vNUMA.

Second, enabling HT will impact the scheduling engine of Hyper-V because it will actually take into account that HT is enabled and facilitates, on a best effort basis, a sort of virtual HT to the VM. In a high demanding environment, this can lead to unpredictable performance on a positive or negative way. You can read more about the way scheduling works in the Hypervisor [Top Level Functional Specification v4.0a](http://download.microsoft.com/download/A/B/4/AB43A34E-BDD0-4FA6-BDEF-79EEF16E880B/Hypervisor%20Top%20Level%20Functional%20Specification%20v4.0.docx){:target="_blank"} published by Microsoft.

And last, HT will impact the way the RSS / VMQ indirection table is build up because when enabled, you have to skip the HT LPs in your assignments (you should therefore enable or disable HT on all your hosts to have a consistent configuration).

The real answer to enable HT or not will depend on the type of workloads and demand you will be running. This is in no way intended to be the Holy Grail in deciding to enable HT or not.

These are just my notes, it always depends!

HTH Ben