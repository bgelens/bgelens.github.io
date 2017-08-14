---
title:  "BYO-DSC with VM Roles (A VM DSC Extension alternative for WAPack)"
date:   2015-04-03 12:00:00
categories: ["WAPACK","Windows Azure Pack","Desired State Configuration","PSDSC"]
tags: ["WAPACK","Windows Azure Pack","PowerShell","VM Role","PSDSC","Desired State Configuration"]
---

This blog post is written and published on Hyper-V.nu: [https://hyper-v.nu/archives/bgelens/2015/04/byo-dsc-with-vm-roles-a-vm-dsc-extension-alternative-for-wapack/](https://hyper-v.nu/archives/bgelens/2015/04/byo-dsc-with-vm-roles-a-vm-dsc-extension-alternative-for-wapack/){:target="_blank"}

# Introduction

The Azure Pack IaaS solution is awesome, we can provide our tenants with a lot of pre-baked solutions in the form of VM Roles.
Tenant users can deploy these solutions without needing the knowledge on how to build the solutions themselves which is a great value add.

But not all customers want pre-baked solutions. Some customers will want to bring their own configurations / solutions with them and they don’t want your pre-baked stuff for multiple reasons (e.g. don’t comply with their standards or requirements). In Azure these customers can make use of the VM extensions. One of the missing pieces of tech in the Azure Pack / SCVMM IaaS solution. It is at this time, very difficult to empower tenant users to bring their own stuff.

In Azure we have a lot of VM extensions available, today I’m going to implement functionality in a VM Role which will behave similarly to the Azure DSC extension (As you probably know by now, I like DSC a lot).

Please note! The implementation will serve as a working example on how you could do this. If you have any questions, please ask them but I will not support the VM Role itself.

# Scenario

A tenant user wants to deploy configurations to their VMs themselves. As a configuration mechanism, the tenant user has chosen Desired State Configuration (DSC). If by any means possible, they want a similar approach in Azure as on your Azure Pack service.

In Azure you can zip your PowerShell script containing your DSC configuration together with the DSC resources it requires. This archive is then uploaded to your Azure Blob storage. The VM DSC Extension picks this archive file up, unpacks it and runs the configuration in the script to generate the MOF file. During this procedure the extension will take in configuration data and take user provided arguments into account.

Check out [http://blogs.msdn.com/b/powershell/archive/2014/08/07/introducing-the-azure-powershell-dsc-desired-state-configuration-extension.aspx](http://blogs.msdn.com/b/powershell/archive/2014/08/07/introducing-the-azure-powershell-dsc-desired-state-configuration-extension.aspx){:target="_blank"} if you are seeking to use DSC in Azure with the VM DSC Extension.

In our Azure Pack VM Role implementation we will try to mimic all this by letting the tenant user zip up its configuration script together with the DSC resources in the same way as they are used to with the Azure DSC extension. In fact, we will use the Azure PowerShell module to do this. Then, because we don’t have blob storage in our implementation (yet), we assume the tenant user has a web server in place where the tenant user will make this zip file available. On this same location a PSD1 file could be published containing DSC configuration data. Also, the VM Role will take arguments as well.

# Prepare a configuration

First let’s create a configuration archive (ZIP file) and a configuration data file (PSD1 file). Then we will stage everything on a web server.

First Create a configuration script for demo purposes.

```powershell
configuration MyNewDomain {
    param (
        [String] $DomainName,
        [String]$SafeModePassword
    )
    $secpasswd = ConvertTo-SecureString $SafeModePassword -AsPlainText -Force
    $SafemodeAdminCred = New-Object System.Management.Automation.PSCredential ("TempAccount", $secpasswd)

    Import-DscResource -ModuleName xActiveDirectory

    node localhost {
        WindowsFeature ADDS {
            Name =  'AD-Domain-Services'
            Ensure = 'Present'
        }

        xADDomain MyNewDomain {
            DomainName = $DomainName
            SafemodeAdministratorPassword = $SafemodeAdminCred
            DomainAdministratorCredential = $SafemodeAdminCred # used to check if domain already exists. Domain Administrator will have password of local administrator
            DependsOn = '[WindowsFeature]ADDS'
        }
    }
}
```

The configuration will produce a MOF file which will make sure that the AD-Domain-Services role is installed and then promote the VM to be the first domain controller of a new domain. The domain name and safe mode password are defined as a parameter and thus can be user configurable at MOF file generation time. The password is taken as a string and then converted to a PowerShell credential object. The configuration needs the xActiveDirectory DSC Module and therefore, this module must be packaged up with the script.

When you create the script, you need to have the DSC resources installed for 2 reasons:

* For IntelliSense to work properly in the ISE. If you don’t install the DSC resource, ISE will red curly brace every configuration item from a non-default DSC resource. So you cannot verify if the syntax and properties are typed correct.
* We will use the Azure PowerShell module to create the archive and it needs the DSC resources to be installed to package them up into the archive.

If you don’t have the xActiveDirectory resource installed on your system and you use WMF5 preview you can simply open a PowerShell console and type: Find-DscResource -moduleName xActiveDirectory | Install-Module -Force -Verbose
When you run WMF4, you need to download and install it manually. For instructions see: [https://gallery.technet.microsoft.com/scriptcenter/xActiveDirectory-f2d573f3](https://gallery.technet.microsoft.com/scriptcenter/xActiveDirectory-f2d573f3){:target="_blank"}

# Prepare the configuration archive

Now we have the configuration script, we need to package it up into an archive. We use the Azure PowerShell module to do this so we have a consistent archive we could use with both the Azure VM DSC extension and the VM Role.
If you don’t have the Azure PowerShell module installed, just install the Microsoft Web Platform Installer and install it from there [http://www.microsoft.com/web/downloads/platform.aspx](http://www.microsoft.com/web/downloads/platform.aspx){:target="_blank"}.

Run the following script from within the directory you saved the configuration script.

```powershell
Import-Module azure
$ConfigPath = '.\BringyourownDC.ps1'
Publish-AzureVMDscConfiguration -ConfigurationPath $ConfigPath -ConfigurationArchivePath "$ConfigPath.zip" -Force
```

It will produce a ZIP file called BringyourownDS.ps1.zip which contains the configuration script and the xActiveDirectory DSC resource.

# Prepare Configuration Data

When you start working with DSC you eventually will get familiar with the concept of separating configuration data from the configuration script. This concept will make configuration scripts re-usable for other nodes as you will define node specifics separately from the configuration script.
If you are not familiar with this concept, please look here: [https://technet.microsoft.com/en-us/library/dn249925.aspx](https://technet.microsoft.com/en-us/library/dn249925.aspx){:target="_blank"}

I won’t go into too much details about this concept here because then I would dwell into another 10 part blog series :-) But we need the configuration data for this blog post to be stored separately from the configuration script as it is the only way to inform PowerShell how we handle credentials in the MOF file (encrypted using certificates or unencrypted).

For this blog post I won’t go into encryption of sensitive data in the MOF file. If you want to read more about that, please look at my previous post here: [http://bgelens.nl/integrating-vm-role-with-desired-state-configuration-part-7-creating-a-configuration-document-with-encrypted-content/](http://bgelens.nl/integrating-vm-role-with-desired-state-configuration-part-7-creating-a-configuration-document-with-encrypted-content/){:target="_blank"}. In this post we will allow unencrypted sensitive data to be present so we won’t have to bother with certificates.

Save the following configuration data to a file named BringyourownDC.psd1.

```powershell
@{
    AllNodes = @(
        @{
            NodeName = "localhost"
            PSDscAllowPlainTextPassword = $true
        }
    )
}
```

If you are planning to deliver this BYO-DSC VM Role to your customers, you would probably want to make some extra effort to enable encryption. In Azure for example, a machine certificate is generated for every VM deployment and the DSC extension uses this certificate to encrypt the sensitive data in the MOF file. You could of course use a similar approach.

# Staging the files on a webserver

Now we both have the archive and the configuration data file. We need to publish them on a webserver. In my test environment I installed a VM with IIS and put both files (BringyourownDC.psd1 and BringyourownDC.ps1.zip) in C:\inetpub\wwwroot.

![](/images/2015-04/BG_BYODSCP_P01.png)

Then I added the PSD1 file extension to the IIS known MIME types to allow IIS to serve psd1 files (it will not do this by default). I used the mime type text/plain.

![](/images/2015-04/BG_BYODSCP_P02.png)

![](/images/2015-04/BG_BYODSCP_P03.png)

# VM Role

As I handled creating VM Roles in great detail in my previous blog posts, I’ll stick to the bare minimum you really need to know about this VM Role. At the bottom of this blog post, you will find a link to my GitHub repository so you can download it and look in the VM Role authoring tool how things are setup.

We add the following script (**BringYourOwnDSC.ps1**) as a script application to the resource extension.

```powershell
param (
    [Parameter(Mandatory)]
    [System.String] $ConfigurationArchiveURL,

    [Parameter(Mandatory)]
    [System.String] $ConfigurationName,

    [System.String] $ConfigurationDataURL,

    [System.String] $ConfigurationArguments
)
$ErrorActionPreference = 'Stop'
$DSCDir = New-Item -Path c:\ -Name 'BringYourOwnDSC' -ItemType Directory -Force
Write-Output -InputObject $ConfigurationArguments
#region download zip
[System.String] $ArchiveName = $ConfigurationArchiveURL.split('/')[-1]
try {
    Write-Output -InputObject "Downloading Configuration Archive $ArchiveName using URL: $ConfigurationArchiveURL"
    Invoke-WebRequest -Uri $ConfigurationArchiveURL `
                      -OutFile "$($DSCDir.PSPath)\$ArchiveName" `
                      -Verbose
    Write-Output -InputObject "Successfully downloaded Configuration Archive"
} catch {
    Write-Error -Message "Failed Downloading Configuration Archive $ArchiveName using URL: $ConfigurationArchiveURL" -Exception $_.exception
    throw $_
}
#endregion download zip

#region download configdata psd1
if ($ConfigurationDataURL -ne 'NA') {
    Write-Output -InputObject "ConfigurationData URL specified. Attempting download"
    $ConfigDataExits = $true
    [System.String] $ConfigDataName = $ConfigurationDataURL.split('/')[-1]
    try {
        Write-Output -InputObject "Downloading Configuration Data $ConfigDataName using URL: $ConfigurationDataURL"
        Invoke-WebRequest -Uri $ConfigurationDataURL `
                          -OutFile "$($DSCDir.PSPath)\$ConfigDataName" `
                          -Verbose
        Write-Output -InputObject "Successfully downloaded Configuration data"
    } catch {
        Write-Error -Message "Failed Downloading Configuration data $ConfigDataName using URL: $ConfigurationDataURL" -Exception $_.exception
        throw $_
    }
} else {
    Write-Output -InputObject "ConfigurationData URL not specified"
    $ConfigDataExits = $false
}
#endregion download configdata psd1

#region unzip and install modules
Unblock-File -Path "$($DSCDir.PSPath)\$ArchiveName"
Expand-Archive -Path "$($DSCDir.PSPath)\$ArchiveName" -DestinationPath "$($DSCDir.fullname)\$($ArchiveName.trim('.ps1.zip'))" -Force
$Modules = Get-ChildItem -Path "$($DSCDir.PSPath)\$($ArchiveName.trim('.ps1.zip'))\" -Directory
foreach ($M in $Modules) {
    if (Test-Path "C:\Program Files\WindowsPowerShell\Modules\$($M.Name)") {
        Write-Output -InputObject "DSC Resource Module $($M.Name) is already present. Checking if there is a version conflict"
        [version] $NewVersion = ((Get-Content -Path "$($M.PSPath)\$($M.Name).psd1" | Select-String "ModuleVersion").tostring()).substring(16).Replace("'", "")
        [version] $CurVersion = ((Get-Content -Path "C:\Program Files\WindowsPowerShell\Modules\$($M.Name)\$($M.Name).psd1"| Select-String "ModuleVersion").tostring()).substring(16).Replace("'", "")
        if ($NewVersion -ne $CurVersion) {
            Write-Output -InputObject "DSC Resource modules are not the same. Overwriting existing module with delivered module"
            Remove-Item -Path "C:\Program Files\WindowsPowerShell\Modules\$($M.Name)" -Recurse -Force
            Copy-Item -Path $M.PSPath -Destination 'C:\Program Files\WindowsPowerShell\Modules' -Recurse
        } else {
            Write-Output -InputObject "DSC Resource Module versions are the same"
        }
    } else {
        Copy-Item -Path $M.PSPath -Destination 'C:\Program Files\WindowsPowerShell\Modules' -Recurse
    }
}
#endregion unzip and install modules

#region call configuration and compile mof

#dot source configuration
Set-Location $DSCDir.fullname
Get-ChildItem -Path .\$($ArchiveName.trim('.ps1.zip')) -Filter *.ps1 | % {
    . $_.fullname
}
if ($ConfigDataExits) {
    $Params = @{}
    $LoadConfigData = get-content .\$ConfigDataName -raw
    $Configdata = & ([scriptblock]::Create($LoadConfigData))
    $Params += @{ConfigurationData=$Configdata}
}
if ($ConfigurationArguments -ne 'NA') {
    if (!($Params)) {
        $Params = @{}
    }
    $splithash = $ConfigurationArguments.split(';')
    $confighash = $splithash | Out-String | ConvertFrom-StringData
    $Params += $confighash
    $params
    write-output ""
    $params.ConfigurationData
    write-output ""
    $params.ConfigurationData.AllNodes
}
if ($Params) {
    try {
        & $ConfigurationName @Params
    } catch {
        $_.exception.message
        Write-Error -Message "Failed compiling MOF with additional parameters" -Exception $_.exception
        throw $_
    }
} else {
    try {
        & $ConfigurationName
    } catch {
        Write-Error -Message "Failed compiling MOF without additional parameters" -Exception $_.exception
        throw $_
    }
}
#endregion call configuration and compile mof

#region configure LCM
[DSCLocalConfigurationManager()]
Configuration LCM {
    Settings {
        RefreshMode = 'Push'
        RebootNodeIfNeeded = $true
        ConfigurationMode = 'ApplyAndAutoCorrect'
    }
}
LCM
Set-DscLocalConfigurationManager -Path .\LCM
#endregion configure LCM

#region start dsc config
Start-DscConfiguration -Path .\$ConfigurationName -Force
#endregion start dsc config
```

This script takes a few parameters:

* **ConfigurationArchiveURL**: The URL where the tenant user has uploaded /staged his archive file (ZIP).
* **ConfigurationName**: The name of the configuration inside the configuration script (MyNewDomain).
* **ConfigurationDataURL**: The URL where the tenant user has uploaded / staged his configuration data file (PSD1).
* **ConfigurationArguments**: Allow the user to specify a semicolon separated list of key value pairs which are used to provide arguments to the configuration parameters (in this case DomainName and SafeModePassword which will be provided by the tenant user in the format: ‘DomainName=Demo.Lab; SafeModePassword=PassWord01’)

The script handles the following:

* It will create a new working folder under the root of C:\ called BrinYourOwnDSC
* It will output the user defined configuration arguments to the output stream so these arguments are captured by the VM Role deployment log.
* At the download ZIP region: it will try to download the archive file. If it does not succeed, a terminating error is produced and deployment is failed.
* At the download configdata psd1 region: it will check if the user provided a URL for configuration data or entered ‘NA’ if the user did not want to add configuration data. If the user did specify a URL, it will attempt to download the PSD1 file. If this would fail, a terminating error is thrown and deployment is failed.
* At the region unzip and install modules: It will unblock and unzip the downloaded archive file into the working directory created earlier. The it will check if it needs to install DSC resources for all resources which were present in the archive by:
  * verifying if the resource is not already present. If it is not, then the resource is installed.
  * If it is present, a version check is done. When the versions are not equal then the version which is already installed is overwritten.
* At the call configuration and compile mof region: it will dot source the configuration script into memory of the current PowerShell process. This makes it available to be called. If the tenant user provided a configuration data file, it will be read and stored in memory so it can be referenced by the params hashtable. If the tenant user provided arguments, they will be added to the params hashtable as well.Then the content of the params hashtable will be written to the output stream so the content can be investigated in the VM Role deployment log. Finally, the configuration is called with the params hastable splatted against it to populate all parameters with values (for more info about splatting: [https://technet.microsoft.com/en-us/library/jj672955.aspx](https://technet.microsoft.com/en-us/library/jj672955.aspx){:target="_blank"}). This results in a MOF file.
* At the configure LCM region: it will produce and apply a meta-configuration MOF file to configure the LCM. The LCM will be put in ‘ApplyAndAutoCorrect’ mode and is allowed to reboot when necessary.
* At the start DSC config region: it runs the Start-DscConfiguration cmdlet with the previously generated MOF.

In the VM Role authoring tool, we add a script application and import the BringYourOwnDSC.ps1 script.

![](/images/2015-04/BG_BYODSCP_P04.png)

The Application Script will be configured with the following parameters:

Name|Value
----|-----
Application Payload | ApplicationScript
DeploymentName | BrindYourOwnDSC
Always Reboot | False
ErrorPolicy | FailOnMatch
Executable | %WINDIR%\System32\WindowsPowerShell\v1.0\PowerShell.exe
Parameters | -NonInteractive -NoProfile -ExecutionPolicy Bypass -Command .\BringYourOwnDSC.ps1 -ConfigurationArchiveURL [Param.ConfigurationArchiveURL] -ConfigurationName [Param.ConfigurationName] -ConfigurationDataURL [Param.ConfigurationDataURL] -ConfigurationArguments [Param.ConfigurationArguments]
StandardErrorPath | C:\VMRole\Log\BYODSC-Error.txt
StandardOutputPath | C:\VMRole\Log\BYODSC-Out.txt
TimeOutInSeconds | 300

In the view definition we will have input fields for the tenant user to specify:

* The URL of the configuration archive
* The name of the configuration in the PowerShell script
* The URL of the configuration data data file

DSC arguments

(Note that I prepopulated default values so speed up testing).

![](/images/2015-04/BG_BYODSCP_P05.png)

![](/images/2015-04/BG_BYODSCP_P06.png)

![](/images/2015-04/BG_BYODSCP_P07.png)

![](/images/2015-04/BG_BYODSCP_P08.png)

When the resource definition and extension are finished, we make them available and assign the gallery item to a plan.

# Deploy the VM Role

Now we have everything in place, we can deploy the VM Role to validate if everything is working correctly.

Note that you cannot use @ signs in these input fields. @ signs will make VMM think you are defining a service template parameter (you will get real funky results!). I asked Microsoft if they had run across this situation with CPS, and they did but did not disclose how they solved it yet with me. As soon as I know, I will update this post.

![](/images/2015-04/BG_BYODSCP_P09.png)

When deployment is done, note that the LCM is still busy configuring the node into the desired state. Unfortunately we cannot inform the tenant user of this through the Azure Pack portal.
You can check the LCM status by running Get-DscLocalConfigurationManager and looking at the LCMState property.

When the LCM is finished and the node has been rebooted by it, you now have a new Domain Controller build from your own configuration!

You can find the files I produced for this blog post on my GitHub repo here: [https://github.com/bgelens/BlogItems/tree/master/BYO-DSC](https://github.com/bgelens/BlogItems/tree/master/BYO-DSC){:target="_blank"}

And on my Onedrive here:
[https://onedrive.live.com/redir?resid=7F054A45CE77AA30!684032&authkey=!AIav2VtK4aLj-JE&ithint=folder%2c](https://onedrive.live.com/redir?resid=7F054A45CE77AA30!684032&authkey=!AIav2VtK4aLj-JE&ithint=folder%2c){:target="_blank"}