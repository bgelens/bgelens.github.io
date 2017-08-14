---
title:  "Lessons learned: DSC Pull Security and Integration with WAPack [DUPSUG]"
date:   2015-07-01 12:00:00
categories: ["WAPACK","Windows Azure Pack","Desired State Configuration","PSDSC","SMA"]
tags: ["WAPACK","Windows Azure Pack","PowerShell","VM Role","PSDSC","Desired State Configuration","SMA"]
---

This blog post is written and published on Hyper-V.nu: [https://hyper-v.nu/archives/bgelens/2015/06/lessons-learned-dsc-pull-security-and-integration-with-wapack-dupsug/](https://hyper-v.nu/archives/bgelens/2015/06/lessons-learned-dsc-pull-security-and-integration-with-wapack-dupsug/){:target="_blank"}

# Introduction

Last Dutch PowerShell User Group (DUPSUG) on May 26th I presented a session on end-2-end Secure DSC Pull Services. The demo scripts can be found here: [https://github.com/bgelens/DUPSUG_052015/tree/master/Secure-End-2-End](https://github.com/bgelens/DUPSUG_052015/tree/master/Secure-End-2-End){:target="_blank"} and I have recorded the demo and posted it on youtube for your review.

<iframe width="560" height="315" src="https://www.youtube.com/embed/wznZcvpEvhw?rel=0" frameborder="0" allowfullscreen></iframe>

On top of that, I demoed interaction / integration between components like DSC Web Pull Server, PKI, VM Role, SMA and Hyper-V.

In this blog post I’m going to describe and share the demo pieces I have shown for the integration / interaction demo. It is a build up from the previous 10 part blog series on DSC integration with Windows Azure Pack VM Roles. So if you are missing pieces to follow or prerequisite knowledge, please start reading here: [http://bgelens.nl/integrating-vm-role-with-desired-state-configuration-part-1-introduction-and-scenario/](http://bgelens.nl/integrating-vm-role-with-desired-state-configuration-part-1-introduction-and-scenario/){:target="_blank"}

During this post, links will be provided to download the presentation and the files.

# Scenario

So first off, what do we want to achieve?
We already have a VM Role in the Azure Pack environment which is used to deploy a DSC Web Pull client VM.

During deployment, this VM enrolls an authentication certificate and using this certificate, it authenticates to a web repository where it downloads an encryption certificate. Then the LCM is configured using the user specified configuration id and using the thumbprints of the authentication and encryption certificates for either function respectively.

So at the end of the VM Role deployment, the VM is setup to pull from the Pull Server in a secure way and from the Azure Pack and VMM perspective it is provisioned.

![](/images/2015-07/BG_DUPSUG_01.png)

![](/images/2015-07/BG_DUPSUG_02.png)

But actually it is not provisioned! We started a new asynchronous process of DSC Pull and convergence which is not monitored by the VMM deployment job.

What will / could happen is that the user could actually interrupt the convergence by stopping the VM or throwing it away and the user might be tempted to do so as well because the VM is not in its desired state yet. So what we want to achieve is to tell the tenant user, the provisioning is actually not finished yet and remove all controls as long as DSC is still converging the node.

Additionally what I want to achieve is use this generic VM Role we use as a DSC Pull client and allow it to become anything by providing install source files on demand. For example, if the VM Role needs install sources for Exchange or SQL, we want SMA to attach a VHDX containing those resources on demand. We use a VHDX for the source files instead of a UNC path as we don’t want to connect to the VM directly over the network because Network Virtualization could be in use. Besides this, the VM could be workgroup / domain joined or in an environment not easily suitable for accessing central file shares. We do have access at the hypervisor level which makes attaching a VHDX the most logical choice.

# The problem

Azure Pack considers the Cloud Service in which the VM Role gets deployed to be provisioned as soon as the VMM deployment job is finished. Natively we don’t have any way of changing this behavior. Luckily we can go into the VMM database and modify some table content by which we can manipulate the cloud service status as experienced by the tenant user. **WARNING**: This is probably not supported by Microsoft!

# The solution

One of my presentation slides walked through the process of what we want to achieve.

![](/images/2015-07/BG_DUPSUG_03.png)

What you see here from the left:

* The tenant user deploys a VM Role
* Azure Pack API calls SPF
* SPF spins up 2 asynchronous processes.
  * The VMM deployment process
  * Triggers an SMA runbook if configured correctly

Let’s zoom in on the VMM deployment process.

* The VM deployment job runs and it will execute the resource extension payload (applications, scripts, etc).
* The job is finished.
* The cloud service is provisioned.

So during the resource extension run, the LCM is configured and the initial pull is triggered actually creating a 3th asynchronous process which is not monitored.

So what can we do? We use that runbook trigger to spin up a runbook to monitor the VMM deployment job. As soon as it is finished, the runbook will modify the VMM database and overwrites the provisioned state of the cloud service to a provisioning state again. The tenant user can then not interrupt the convergence process anymore!

Next the SMA process monitors the VM from the Hyper-V level until an expected key value pair will show up through the VM guest integration component of key value pair exchange. The value we are waiting for will tell SMA what install source disk needs to be attached or if this is not required. In the DSC configuration we make sure that this key value pair will always be configured either with a value targeting an install source or with a value of ‘none’ telling SMA to skip this process. KVP exchange values are configured in the VMs registry.

The DSC configuration will make use of a resource which waits for the install source disk to arrive and then assigns it a drive letter. Another resource will install what is desired and finally the final registry key is set to tell SMA the LCM is finished with convergence. As soon as this is picked up, the cloud service is set into provisioned state again making it accessible for the tenant user.

If something would fail during this entire process, SMA would also be able to set the cloud service into a failed state.

# Implementation

In this case we will setup the VM Role to be deployed as a SQL server.

## Compile the MOF file

The configuration script generating the MOF file:

```powershell
Configuration SQLServerInstall {
    param (
        [String] $Node,

        [PSCredential] $Credential
    )
    Import-DscResource -Module xdisk, xsqlserver

    node $node {
        Registry InstallDiskKVP {
            Key = 'HKLM:\SOFTWARE\Microsoft\Virtual Machine\Guest'
            ValueName = 'InstallDisk'
            ValueData = 'SQL.vhdx'
            ValueType = 'String'
            Ensure = 'Present'
        }

        User LocalAdmin {
            UserName = $Credential.UserName
            Ensure = 'Present'
            Password = $Credential
            Description = 'User created by DSC'
            PasswordNeverExpires = $true
            PasswordChangeNotAllowed = $true
        }

        Group Administrators {
            GroupName = 'Administrators'
            MembersToInclude = $Credential.UserName
            DependsOn = '[User]LocalAdmin'
        }

        WindowsFeature DotNet35 {
            Ensure = 'Present'
            Name = 'NET-Framework-Features'
        }

        xWaitForDisk SourceDisk {
            DiskNumber = 1
            RetryCount = 720
        }

        xDisk SourceDisk {
            DiskNumber = 1
            DriveLetter = 'I'
            DependsOn = '[xWaitForDisk]SourceDisk'
        }

        xSqlServerSetup InstallSqlServer {
            InstanceName =  'DSCInstance'
            SourcePath = 'I:'
            SourceFolder = '\SQL2014'
            Features= 'SQLENGINE'
            SecurityMode = 'SQL'
            UpdateEnabled = $false
            SQLSysAdminAccounts = $Credential.UserName
            SetupCredential = $Credential
            SAPwd = $Credential
            DependsOn = '[xDisk]SourceDisk','[Group]Administrators'
        }

        xSQLServerFirewall SetSQLFirewall {
            Ensure = 'Present'
            SourcePath = 'I:'
            SourceFolder = '\SQL2014'
            Features= 'SQLENGINE'
            InstanceName = 'DSCInstance'
            DependsOn = '[xSqlServerSetup]InstallSqlServer'
        }

        Registry LCMStatusKVP {
            Key = 'HKLM:\SOFTWARE\Microsoft\Virtual Machine\Guest'
            ValueName = 'LCMStatus'
            ValueData = 'Finished'
            ValueType = 'String'
            Ensure = 'Present'
            DependsOn = '[xSQLServerFirewall]SetSQLFirewall'
        }
    }
}
```

The configuration does the following things:

* Creates a registry key which handles the KVP exchange to tell SMA which install source disk to couple
* Create a new local admin account
* Ensures .Net 3.5 is installed
* Waits for SMA to attach the install source disk
* Assigns a drive letter to the install source disk
* Install SQL Server 2014 from the install source disk
  * Adds new local admin to SQL SysAdmin role
  * Configures SA password to be the same as the new local admin
* Opens Windows Firewall ports for remote SQL management
* Creates a registry key which handles the KVP exchange to tell SMA the node is in its desired state.

As you can see, this configuration requires two non-native DSC resource modules xDisk and xSQLServer.

If you are using your DSC pull server as the authoring station and is running WMF5, you could use the following script to install all required modules and have them available in the DSC Web Pull Module repository as well.

```powershell
Find-DscResource | Select-Object -Property ModuleName -Unique | %{
    if (!(Get-Module $_.modulename -ListAvailable)) {
        Install-Module $_.ModuleName -Force
    }
}
$PullServerModulesPath = 'C:\Program Files\WindowsPowerShell\DscService\Modules'
Get-DscResource | Select-Object -Property Module -Unique | %{
    if ($_.Module -match '^[C,X]') {
        $Module = Get-Module $_.Module -ListAvailable
        $Manifest = Test-ModuleManifest $Module.Path
        $ArchiveName = $Manifest.Name + '_' + $Manifest.Version.ToString() + '.zip'
        if (!(Test-Path "$PullServerModulesPath\$ArchiveName")) {
            Write-Verbose -Message "Adding $($Manifest.name) with version $($Manifest.Version.ToString()) to DSC Pull Server Modules repository" -Verbose
            #clear readonly (compress archive seems to break with access denied if this is set)
            Get-ChildItem $Module.ModuleBase -Recurse | %{attrib.exe -R $_.FullName}
            Compress-Archive -Path $module.ModuleBase -DestinationPath "$PullServerModulesPath\$ArchiveName" -Force
            New-DSCCheckSum -Path $PullServerModulesPath\$ArchiveName -Force
        }
    }
}
```

Now you have everything in place, you can compile the MOF file and generate the checksum. First generate the configuration Id and PFX for the certificate used for encryption by using the PFX creation script you can find here: [http://bgelens.nl/integrating-vm-role-with-desired-state-configuration-part-7-creating-a-configuration-document-with-encrypted-content/](http://bgelens.nl/integrating-vm-role-with-desired-state-configuration-part-7-creating-a-configuration-document-with-encrypted-content/){:target="_blank"}

Next load the configuration into memory and then run the following code to compile the MOF file. Note that you need to replace the GUID at the node variable by the one you generated earlier.

```powershell
$Node = '2a8a4e1e-84a9-446b-ba66-38f85c924d14'
$certfile = "C:\PublicCerts\$Node.cer"
$LocalAdminCred = (Get-Credential -UserName 'DSCAdmin' -Message 'Enter password for DSCAdmin')

$ConfigData = @{
    AllNodes = @(
        @{
            NodeName = $Node
            CertificateFile=$certfile
        }
    )
}

SQLServerInstall -Node $Node `
                 -OutputPath 'C:\Program Files\WindowsPowerShell\DscService\Configuration' `
                 -ConfigurationData $ConfigData `
                 -Credential $LocalAdminCred

New-DSCCheckSum -ConfigurationPath "C:\Program Files\WindowsPowerShell\DscService\Configuration\$Node.mof" -Force
```

## Setup SMA runbooks

In a previous post I already described how to setup SMA runbooks to be triggered when a VM Role gets deployed. You can read about it here: [http://bgelens.nl/designing-vm-roles-with-sma-in-mind/](http://bgelens.nl/designing-vm-roles-with-sma-in-mind/){:target="_blank"} Because of this, I won’t go into detail in this post.

You can download the SMA runbooks I used during my presentation from my GIT repo here: [https://github.com/bgelens/DUPSUG_052015/tree/master/WAPack-Integration](https://github.com/bgelens/DUPSUG_052015/tree/master/WAPack-Integration){:target="_blank"}

Runbooks:

Name|Description
----|-----------
TriggerSPF-CreateVMRoleDSC | This is the master runbook and needs to be configured with the SPF tag so you can assign it to the Create VM Role trigger. It will run all child runbooks in the correct order.
Get-VMRoleVMs | This child runbook will acquire the VMM VM objects associated with the VM Role (a VM role can be of 1 or multiple instances).
Wait-VMMJob | This child runbook will wait for the VMM job to finish
Set-CloudServiceStatus | This child runbook is used to manipulate the VMM database so the Cloud Service will be in either privisioning, provisioned or failed state.
Wait-VMKVPValue | This child runbook is used to monitor the VM with Hyper-V Key Value Pair exchange.
Add-InstallSourceDisk | This child runbook will create a new diff disk from the install sources disk and attaches it to the VM.
Stop-LingeringSession | This child runbook is used to terminate remoting sessions which are created by inlinescript activities. This is a workarround to deal with the using scope bug: [https://connect.microsoft.com/PowerShell/feedback/details/894721/workflows-using-multiple-inlinescripts-with-the-pscomputername-does-not-get-correct-variables-from-the-workflow-when-using-using-scoping](https://connect.microsoft.com/PowerShell/feedback/details/894721/workflows-using-multiple-inlinescripts-with-the-pscomputername-does-not-get-correct-variables-from-the-workflow-when-using-using-scoping){:target="_blank"}

Two SMA assets need to be created:

Name | Type | Value
-----|------|------
SCVMM Service Account | Credential | Credentials which have admin rights on VMM, Hyper-V and have DBO owner rights on VMM database
VMMServer | String | VMM Server or Cluster name

Let’s zoom into some of the runbooks so you have a better understanding of what is going on (please keep in mind that I developed these runbooks specifically for the DUPSUG session which is why there are not a lot off comments in there).

### Set-CloudServiceStatus runbook

The runbook dealing with reconfiguring the cloud service state will change the SQL table governing the state by talking to SQL via the VMM server. We do this to consolidate the actions involved or else we would be remoting all over the place. It turns out that if you connect to your VMM server via the cluster name, you cannot use CredSSP as it reverse checks the computer name and finds a mismatch. We need CredSSP to handle the second hop so first we will resolve the Active Node name in case you have a clustered VMM setup.

![](/images/2015-07/BG_DUPSUG_04.png)

>> Note that you will see the runbook Stop-LingeringSession being called a lot. This runbook is called inline and actually will terminate the remote PowerShell session via a CIM connection. I do this, because in WMF4 there is a bug with running inlinescript activities. The bug breaks the using variable scope if 2 or more inlinescript activies connect to the same remote computer within a few minutes because the remoting sessions will linger. For more info see: [https://connect.microsoft.com/PowerShell/feedback/details/894721/workflows-using-multiple-inlinescripts-with-the-pscomputername-does-not-get-correct-variables-from-the-workflow-when-using-using-scoping](https://connect.microsoft.com/PowerShell/feedback/details/894721/workflows-using-multiple-inlinescripts-with-the-pscomputername-does-not-get-correct-variables-from-the-workflow-when-using-using-scoping){:target="_blank"}

Next the runbook will connect to the active VMM node and based on the parameter input (Provisioning, Provisioned or Failed) will update the SQL table row corresponding to the cloud service accordingly. When it has updated the row, the runbook calls the Set-CloudService cmdlet with the -RunREST switch which will actively notify SPF of the state change cascading it up the Azure Pack stack so the user actually experiences the update to have taken place.

![](/images/2015-07/BG_DUPSUG_05.png)

### Wait-VMKVPValue runbook

This runbook monitors for Guest initiated Hyper-V key value pair updates. A CIM session is established from the SMA runbook worker to the Hyper-V host. Then a function is loaded into memory which will be called repeatedly until a key will show up or a key with a specified value will show up. A failsafe of 30 minutes is built in which was suitable for my demo purposes. If you are looking to adopt this runbook then maybe it’s better to parametrize this value so you could define a value from the parent per case.

![](/images/2015-07/BG_DUPSUG_06.png)

### Add-InstallSourceDisk runbook

This is the runbook which attached the VHDX files if a DSC configuration needs additional install sources. As you can see, in this case a differencing disk is created from a source disk which is located on the Hyper-V host itself. Again, this was very suitable for demo purposes but if you are looking to adopt you would probably do this from a file share or via another means. The differencing disk is attached to the VM’s SCSI adapter.

![](/images/2015-07/BG_DUPSUG_07.png)

## Changing Azure Pack Tenant Portal refresh

Now you have seen all components in place there is one thing left to configure. The Windows Azure Pack tenant portal has a long refresh interval configured by default. Because of this, once the runbook overwrites the provisioned state with the provisioning state, it could take a long time for the tenant user to notice this. By running this script on your tenant portal machines you will decrease this refresh intervals for a better experience.

```powershell
Unprotect-MgmtSvcConfiguration -Namespace tenantsite
$pspath = 'MACHINE/WEBROOT/APPHOST/MgmtSvc-TenantSite'
Set-WebConfigurationProperty -PSPath $pspath -Filter "appSettings/add[@key='Microsoft.Azure.Portal.Configuration.PortalConfiguration.DataSetNormalPollingIntervalInSeconds']" -name "value" -value "10"
Set-WebConfigurationProperty -PSPath $pspath -Filter "appSettings/add[@key='Microsoft.Azure.Portal.Configuration.PortalConfiguration.DataSetSlowPollingIntervalInSeconds']" -name "value" -value "10"
Set-WebConfigurationProperty -PSPath $pspath -Filter "appSettings/add[@key='Microsoft.Azure.Portal.Configuration.OnPremPortalConfiguration.DataSetLongPollingIntervalInSeconds']" -name "value" -value "60"
Protect-MgmtSvcConfiguration -Namespace tenantsite
```

# Closing notes

I had a lot of fun trying to integrate all these components which are not natively attuned to each other. Luckily, in Azure things are rapidly moving forward and DSC is actually a first class citizen there. Now that Azure Stack has been announced, we will also get the opportunity of running VM extensions on premises in our private clouds which will fill in the integration gap through native means. I’m really looking forward to working with ARM in combination with DSC and maybe there will be a good management solution for DSC there as well. Azure Automation DSC is already looking very promising on this aspect.

Hope you have enjoyed my session if you were at DUPSUG and if you were not, I hope this blog post has given you enough reference of what was presented. If any questions arise, please post a comment or find me on twitter! [@bgelens](https://twitter.com/bgelens){:target="_blank"}