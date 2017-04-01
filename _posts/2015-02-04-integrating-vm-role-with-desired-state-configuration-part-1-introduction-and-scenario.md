---
title:  "Integrating VM Role with Desired State Configuration Part 1 – Introduction and Scenario"
date:   2015-02-04 12:00:00
categories: ["WAPACK","Windows Azure Pack","Desired State Configuration","PSDSC"]
tags: ["WAPACK","Windows Azure Pack","PowerShell","VM Role","PKI","PSDSC","Desired State Configuration"]
---

This blog post is written and published on Hyper-V.nu: [https://hyper-v.nu/archives/bgelens/2015/02/integrating-vm-role-with-desired-state-configuration-part-1-introduction-and-scenario/](https://hyper-v.nu/archives/bgelens/2015/02/integrating-vm-role-with-desired-state-configuration-part-1-introduction-and-scenario/){:target="_blank"}

Series Index:
1. [Introduction and Scenario](http://bgelens.nl/integrating-vm-role-with-desired-state-configuration-part-1-introduction-and-scenario/)
2. [PKI](http://bgelens.nl/integrating-vm-role-with-desired-state-configuration-part-2-pki/)
3. [Creating the VM Role (step 1 and 2)](http://bgelens.nl/integrating-vm-role-with-desired-state-configuration-part-3-creating-the-vm-role-step-1-and-2/)
4. [Pull Server](http://bgelens.nl/integrating-vm-role-with-desired-state-configuration-part-4-pull-server/)
5. [Resource Kit](http://bgelens.nl/integrating-vm-role-with-desired-state-configuration-part-5-resource-kit/)
6. [PFX Repository Website](http://bgelens.nl/integrating-vm-role-with-desired-state-configuration-part-6-pfx-repository-website/)
7. [Creating a configuration document with encrypted content](http://bgelens.nl/integrating-vm-role-with-desired-state-configuration-part-7-creating-a-configuration-document-with-encrypted-content/)
8. [Create, deploy and validate the final VM Role](http://bgelens.nl/integrating-vm-role-with-desired-state-configuration-part-8-create,-deploy-and-validate-the-final-vm-role/)
9. [Create a Domain Join DSC resource](http://bgelens.nl/integrating-vm-role-with-desired-state-configuration-part-9-create-a-domain-join-dsc-resource/)
10. [Closing notes](http://bgelens.nl/integrating-vm-role-with-desired-state-configuration-part-10-closing-notes/)
# Introduction

I started this blog series because I strongly believe in Desired State Configuration (DSC) and find the lack of native support / exposure in the vCurrent System Center suite a bit disappointing.

When System Center 2012 R2 was announced, and specifically SCVMM,  there was this emphasis on DSC support being available. I never figured out what was actually meant with this statement however. I just could not seem to find any native way of integrating with DSC.

I scavenged through channel 9 and eventually rediscovered this session: D[esired State Configuration in Windows Server 2012 R2 PowerShell](http://channel9.msdn.com/events/TechEd/NorthAmerica/2013/MDC-B302#fbid=){:target="_blank"}

Here at 44 minutes in, DSC integration is shown as part of the VM Role resource extension.

![](/images/2015-02/BG_VMRole_DSC_P01P001.png)

Not much integration if you ask me because it’s up to the scripting skills and creativity of the one developing the Resource Extension and not provided natively at the VMM level.

In this blog series I will look at how to target VM Roles at the DSC Pull server. Why choose the Pull Server scenario you ask? I figure the DSC Pull server is a crucial part of an Enterprise DSC adoption / implementation / strategy (controlled deployments instead of ad-hoc). Since I work mostly at Enterprise customers, it make sense to look at the Pull server.

For this blog series I imagine a company to have a DSC deployment policy in place which dictates a few rules:

* No ad-hoc (Start-DscConfiguration) DSC configurations are allowed in production.
* Making use of controlled releases for production systems via DSC Pull server is mandatory.
* Configuration documents (MOF files) are not allowed to have unencrypted sensitive information.
* Local Configuration Manager (LCM) instances are allowed to connect with the DSC Pull server only if they are trusted / authenticated.

Why have this imaginary policy? Because first, I think my customers want release management to be in place. Minimizing risk for production environments always has high priority. Second, I know my customers want a decent level off security for their environment.

Customers are evolving their delivery methods to be agile. They want to adopt DevOps to better support there delivery methods but at the same time these customers have issues letting go of their well-established processes (ITIL). This is where the DSC Pull server infrastructure comes into play. DSC configurations are developed in a DEV environment using either another Pull Server or ad-hoc configurations. When fully tested, these configurations will be made available via the Pull server for production systems (in other words: controlled releases).

Some discussion is going on momentarily about the Local Configuration Manager Configuration IDs being used as if they are security principals but aren’t suitable for the task (see: link). In my opinion this is happening because documentation and example scenario’s to configure the LCM with certificate based authentication is still missing or is very scarce. That is why in this blog series we go over the implementation of DSC Pull Server with client certificate authentication which guarantees that the Configuration ID is used for its intended purpose (to be a unique configuration document reference ID). Besides certificate based authentication, encryption of the configuration document (MOF file) will be handled in a secure way using certificates as well.

So why use the VM Role to deploy the VM’s and not use a DSC configuration for that? I think (IMO) the DSC resources for Azure and Hyper-V are excellent for building your lab environment. It however seems strange to me to assign a configuration document to an LCM instance which, once it has run the configuration, does not have any relationship with the resources created. I think there is a place for VM deployment automation (VM Role,  Service Template, Azure Resource Manager), orchestration (SMA, Orchestrator, Azure Automation, partly DSC) and end-state instance configuration automation (DSC, PowerShell, WebDeploy).

Besides the secure DSC Pull Server, the required infrastructure to support this scenario will be discussed as well, just as the VM Role resource definition and extension. I plan to eventually build upon this series and extend it with SMA and Azure topics so when it’s done, stay tuned for more!

Note that through this series it is assumed you already have basic knowledge of DSC, DSC Pull Server and Azure Pack IaaS implementations.

# Scenario

For this scenario a VM Role will be created utilizing both the resource definition and resource extension. The resource extension will be used to run scripts at deployment time which will configure the Local Configuration Manager of the VM and provide for all prerequisites.

![](/images/2015-02/BG_VMRole_DSC_P01P002.png)

When the VM Role gets deployed and the resource extension is running the deployment scripts, the following steps will be taken:

* The VM will download the Certificate Authority Root certificate from its Authority Information Access (AIA) location.
* The Root CA certificate will be imported into the computer trusted root certificate store.
* A Client authentication certificate is requested from the certificate authority using the CA’s web enrollment services (the request will be done using credentials provided via the VM Role view definition).
* Using the Client authentication certificate, a download request will be made against a web server hosting certificate PFX containers which are used for configuration document (MOF file) encryption (encrypted is done with the public key and the VM therefore needs the private key to decrypt). The PFX file will have a filename matching the Configuration ID. For this blog series, I assume a PFX is available for every configuration document.
* The PFX file will be imported in the computer certificate store.
* The LCM will be configured targeting the DSC Pull server. As part of its configuration, the client authentication certificate is selected to authenticate against the Pull server and the certificate acquired from the PFX file will be selected as the decryption certificate.

The scenario provides for an end to end secure deployment (solving the chicken and the egg problem) because tenant users are given explicit access to the VM Role Gallery Item through their subscription and are therefore authorized for utilizing DSC Pull services. Also the Client authentication certificate will be requested using validated credentials provided during the VM Role deployment. Lastly the configuration documents credential references / sensitive data are encrypted.

If you want to follow along with this series you’ll have to prepare the following (roles can be collocated on the same VM, I separated as much as my test environment allowed for):

* Active Directory (Domain Controller).
* Hyper-V host
* SCVMM
* SPF
* Windows Azure Pack setup for IAAS
* 1 VM joined for the domain for the Certificate Authority and enrollment services.
* 1 VM (can be joined but doesn’t have to) for the DSC Pull Server and PFX certificate repository.

Next up, the environment will be prepped. Hope you enjoy reading!