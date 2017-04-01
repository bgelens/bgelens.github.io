---
title:  "Integrating VM Role with Desired State Configuration Part 10 – Closing notes"
date:   2015-02-25 12:00:00
categories: ["WAPACK","Windows Azure Pack","Desired State Configuration","PSDSC"]
tags: ["WAPACK","Windows Azure Pack","PowerShell","VM Role","PKI","PSDSC","Desired State Configuration"]
---

This blog post is written and published on Hyper-V.nu: [http://hyper-v.nu/archives/bgelens/2015/02/integrating-vm-role-with-desired-state-configuration-part-10-closing-notes/](http://hyper-v.nu/archives/bgelens/2015/02/integrating-vm-role-with-desired-state-configuration-part-10-closing-notes/){:target="_blank"}

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

So here we are at post number 10, a good number to finalize this series with some closing notes.

I must say, creating this series has taken a lot of energy and time but the reactions I received thus far has made it more than worth it. Thank you community for the great feedback and thank you for reading! I learned a lot writing the series, I hope you did while reading it.

# DSC Resources

Microsoft has made a lot of effort in producing and releasing DSC resources to the public. These resources are great but you must know that the prefixed x stands for experimental. This means you don’t get any support and some of them may not be entirely stable or don’t work out the way you think they should work.

I took xComputer for an example in this blog series. A great resource to domain join your computer (and do a lot more) but not suitable in a Pull Server scenario where the computer name is not known up front.

Is this bad? No, I don’t think so. The xComputer resource was probably built with a certain scenario in mind, and in its intended scenario it works just great. If a resource does not work for you, you could still take a look at it and build your own ‘fork’ or you could start from scratch. The modules / resources are written using PowerShell, so if you can read and write PowerShell, you’re covered. Just be creative and you will manage!

# Pull Server

Ad-hoc configurations are more dynamic then Pull Server delivered configurations. When using ad-hoc you are able to dynamically populate the configuration document content by adding in configuration data which is gathered / generated on the fly. Even the configuration block itself can contain dynamic elements. The resulting MOF file (configuration document) is created on and tailored for its destination. The downside of this approach is that configurations are done on the fly which can turn out into an ‘oops’ situation more quickly.

Pull Server configurations are more difficult to setup because configuration documents are static and created up front. If you create a single configuration for multiple instances (e.g. web server farm), the configuration should be node name agnostic.  The gain here is that configurations are delivered in a more controlled fashion including the required modules. When a configuration change is made, the change can be pulled and implemented automatically.

# Beware of using over-privileged accounts

Beware of over-privileged credentials used in configuration documents.  Although you have taken all necessary precautions by encrypted sensitive data using certificates, if the box gets owned, the certificate private key is exposed and therefore the credentials have fallen in the wrong hands.

For example: Credentials which interact with AD to domain join should be able to do just that. In a VM Role scenario I would build an SMA runbook to pre-create a computer account as soon as the VM Role gets deployed. A low privileged domain account is then delegated control over the object so it is able to domain join. DSC in this case does not have to create a computer account but can just establish the trust relationship.

# VM Role challenges

When dealing with the VM Role and external automation / orchestration, some challenges arise.

There is no (or at least not an easy way) way of coordinating between the VM Role resource extension and DSC configuration state. DSC could potentially reboot the system and go on with configuring after the reboot. It then reboots again and again and again depending on the configuration. The resource extension allows for a script or application to restart the system but treats it as the end of a task. As you don’t know the configuration reboot requirements up front, managing this in the resource extension becomes a pain so you will probably not do this. As a result, the VM Role is provisioned successfully for the tenant user but really is still undergoing configuration.

So VMs will have a provisioned state and become accessible for the tenant user while the VM is still undergoing it’s DSC / SMA configuration. A user can potentially login, shutdown, restart, intervene and thereby disrupt the automation tasks. In case of DSC this is not a big problem as the consistency engine will just keep on going until consistency is reached but if you use SMA for example, well, it becomes a bit difficult.

Another scenario, the user logs in and finds that the configuration he expected is not implemented. Because the user does not know DSC is used, the VM Role is thrown away and the user tries again and again until eventually he is fed up with the service received and starts complaining.

A workaround I use today at some customers is to temporarily assign the VM Role to another VMM user when the VMM deployment job is finished. This removes the VM Role from the tenant users subscription and thereby from their control. The downside here is obvious, the tenant user just experienced a glitch where the VM Role just disappeared and tries to deploy it again. Because the initial name chosen for the Cloud Service is now assigned to another user and subscription, the name is available again so there is a potential for naming conflicts when assigning the finished VM Role back to the original owner.

At this time we cannot lock out the VM controls from the tenant user or manipulate the provisioning state. I added my feedback for more control at the WAPack feedback site here: [http://feedback.azure.com/forums/255259-azure-pack/suggestions/6391843-option-to-make-a-vm-inaccessible-for-tenant-user-a](http://feedback.azure.com/forums/255259-azure-pack/suggestions/6391843-option-to-make-a-vm-inaccessible-for-tenant-user-a){:target="_blank"}.

# Bugs / issues

While developing this series I faced some issues / bugs which I logged at connect:

* Class defined DSC resource module cannot be acquired via Pull Server because of script module structure check of LCM WebDownloadManager       
    [https://connect.microsoft.com/PowerShell/feedback/details/1143212/class-defined-dsc-resource-module-cannot-be-acquired-via-pull-server-because-of-script-module-structure-check-of-lcm-webdownloadmanager](https://connect.microsoft.com/PowerShell/feedback/details/1143212/class-defined-dsc-resource-module-cannot-be-acquired-via-pull-server-because-of-script-module-structure-check-of-lcm-webdownloadmanager){:target="_blank"}
* Private Key not accessible for DSC LCM when key is generated using CNG instead of legacy CSP
    [https://connect.microsoft.com/PowerShell/feedback/details/1110885/private-key-not-accessible-for-dsc-lcm-when-key-is-generated-using-cng-instead-of-legacy-csp](https://connect.microsoft.com/PowerShell/feedback/details/1110885/private-key-not-accessible-for-dsc-lcm-when-key-is-generated-using-cng-instead-of-legacy-csp){:target="_blank"}
* DSC LCM 2.0 Configuration does not work with Client Certificate Authentication
    [https://connect.microsoft.com/PowerShell/feedback/details/1112879/dsc-lcm-2-0-configuration-does-not-work-with-client-certificate-authentication](https://connect.microsoft.com/PowerShell/feedback/details/1112879/dsc-lcm-2-0-configuration-does-not-work-with-client-certificate-authentication){:target="_blank"}

# What’s next?

First I will do a speaker session at the SCUG/Hyper-V.nu [event](http://scug.nl/events/){:target="_blank"} about VM Roles with Desired State Configuration. And no, it will not be a walkthrough of this blog series so much to be done generating the content for this presentation. I think I will blog about the content of my session once I have done it.

Then I will start a new series build upon what we learned in this blog series in the near future. I have many ideas about what could be done but I still have to think about scope and direction for a bit. This series took up a lot more time than I anticipated and I have changed scope many times because I wanted to do just too much. Just for a spoiler for the next series, I know it will involve SMA :-) Stay tuned at [www.hyper-v.nu](www.hyper-v.nu){:target="_blank"}!