---
title:  "SCVMM 2012 R2 UR4, move host group assigned services to a cloud (The supported way)!"
date:   2014-12-03 12:00:00
categories: ["Hyper-V","Windows Server","SCVMM"]
tags: ["Hyper-V","Windows Server","SCVMM"]
---
A while back I posted an item about how to move VMM services (deployed instances of service templates) from host groups to clouds in an unsupported way using some tsql against the VMM database. A link to this post can be found here: [http://wp.me/p1COfr-H](http://wp.me/p1COfr-H){:target="_blank"} Last week I received a comment on this post by David Padley who took my idea and constructed a PowerShell module around it. This script can be found here: [https://onedrive.live.com/redir?resid=3064018F27C72ECC%21113](https://onedrive.live.com/redir?resid=3064018F27C72ECC%21113){:target="_blank"}

Yesterday I had a call with PMs from the VMM product group and I mentioned the lack of this functionality as one of my pain points. I explained that in some cases VMM gets introduced into an environment utilizing host groups only but the implementation will eventually evolve by adding Clouds for self service purposes. During this maturity process of VMM (but before the first cloud gets created) Service templates are used against the host groups. In my experience in Enterprise environments, Service templates are mainly used to deploy Infrastructure components which have a long lifecycle (3+ Years). Eventually a cloud gets created because the company wants to segregate control over the fabric from the controls over VMs (not at the Azure Pack level yet). The company will try to assign VMs to a cloud but the VMs deployed by the Service template cannot be moved. The error message states that if possible to redeploy the Service to the cloud or else let it be in it's current state as it cannot be moved in this manner. This can come as a shocker because this SQL server cannot be removed as it has become a business critical component.

Apparently Microsoft had already listened :) They had another use case where a customer had 1 Cloud with multiple host groups which were geographically dispersed. This company deployed there Service to the host group level to make sure it landed in the correct geographical site and then they needed to move the Service to the Cloud level to facilitate self service (start, stop, etc). I'm happy to tell you Microsoft added functionality as of UR4 which can move deployed services from a host group level to a Cloud using native tooling (supported!). The functionality is not exposed at the GUI level but can be utilized using PowerShell.

```powershell
Get-SCService -Name "Service"| Set-SCService -Cloud (Get-SCCloud -Name "Cloud")
```

As of now I’m only able to move the Service to a Cloud from the host group level but not back again. Also moving between Clouds looks unsupported (I cannot do it, NO_PARRAM error). I guess it’s not implemented in it’s final form yet but this implementation is already a mayor improvement! As a side note:

* fill in surveys,
* get contacted for a lync session
* rant for an hour with PMs
* make another appointment to rant again for an hour

Microsoft listens! It may take a while but if your case is valid, you will be heard.