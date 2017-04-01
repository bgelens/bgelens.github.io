---
title:  "Integrating VM Role with Desired State Configuration Part 5 – Resource Kit"
date:   2015-02-12 12:00:00
categories: ["WAPACK","Windows Azure Pack","Desired State Configuration","PSDSC"]
tags: ["WAPACK","Windows Azure Pack","PowerShell","VM Role","PKI","PSDSC","Desired State Configuration"]
---

This blog post is written and published on Hyper-V.nu: [https://hyper-v.nu/archives/bgelens/2015/02/integrating-vm-role-with-desired-state-configuration-part-5-resource-kit/](https://hyper-v.nu/archives/bgelens/2015/02/integrating-vm-role-with-desired-state-configuration-part-5-resource-kit/){:target="_blank"}

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

In this (relatively short) post the DSC modules from the DSC resource kit will be added to the DSC Pull Server modules repository. This makes the DSC resources available for download by the LCM instances directly from the Pull Server. When an LCM Pulls it’s configuration it parses the configuration for module information. If the LCM finds it is missing modules or has incorrect versions of them, it will try to download them from the Pull Server. If it can’t get them from the Pull Server, configuration will fail.

First let’s download the latest resource kit (v9 at the time of writing). You can find it here: [https://gallery.technet.microsoft.com/scriptcenter/DSC-Resource-Kit-All-c449312d](https://gallery.technet.microsoft.com/scriptcenter/DSC-Resource-Kit-All-c449312d){:target="_blank"}. Save it somewhere reachable for the DSC Pull Server.

# Installing the Resource Kit

We will install the modules for the DSC Pull Server itself so they can be used for creating configuration documents later on. I created a little script to handle this process automatically. Let’s take a look:

```powershell
#requires -version 5
param
(
    [Parameter(Mandatory=$true,
                HelpMessage='Enter a filepath to the Resource Kit Zip File.')]
    [ValidateScript({
                        if ((Test-Path -Path $_) -and ($_.split('.')[-1]) -eq 'zip')
                        {
                            $true
                        }
                        else
                        {
                            $false
                        }
                    })]
    [String]$Path,
        
    [Parameter(HelpMessage='When the Force switch is specified, modules will forcefully be overwritten')]
    [Switch]$Force
)
Process
{
    $ExpandDir = New-Item -Path $env:TEMP -Name ResourceKitExtract -ItemType Directory -Force
    Unblock-File -Path $Path
    Expand-Archive -Path $Path -DestinationPath $ExpandDir.FullName -Force
    $Modules = Get-ChildItem -Path "$($ExpandDir.FullName)\All Resources"
    foreach ($M in $Modules)
    {
        $DestinationPath = "$env:ProgramFiles\WindowsPowerShell\Modules\$($M.Name)"
        #check if module already exists
        if (Test-Path -Path $DestinationPath)
        {
            [version]$NewVersion = ((Get-ChildItem -Path $M.FullName -Filter '*.psd1' | Get-Content | Select-String "ModuleVersion").tostring()).substring(16).Replace("'", "")
            [version]$CurVersion = ((Get-ChildItem -Path $DestinationPath -Filter '*.psd1' | Get-Content | Select-String "ModuleVersion").tostring()).substring(16).Replace("'", "")
            if ($NewVersion -gt $CurVersion)
            {
                Write-Verbose -Message "Module $($M.Name) already exists but is newer in provided resource kit and will be overwritten"
                Write-Verbose -Message "Current Version: $CurVersion"
                Write-Verbose -Message "Resource Kit Version: $NewVersion"
                $overwrite = $true
            }
 
            if ($CurVersion -gt $NewVersion)
            {
                Write-Verbose -Message "Module $($M.Name) already exists but the resource kit version is older than the version on the system" -Verbose
                Write-Verbose -Message "Current Version: $CurVersion" -Verbose
                Write-Verbose -Message "Resource Kit Version: $NewVersion" -Verbose
                Write-Verbose -Message "If you want to overwrite the module, please manually remove the module first: $DestinationPath" -Verbose
            }
 
            if ($overwrite -or $Force -and -not $Newer)
            {
                if ($force)
                {
                    Write-Verbose -Message "Module $($M.Name) will be overwritten as specified by the Force switch"
                }
                Remove-Item $DestinationPath -Force -Recurse
                Copy-Item -Path $M.FullName -Destination $DestinationPath -Force -Recurse
            }
        }
        else
        {
            Write-Verbose -Message "Module $($M.Name) will be added"
            Copy-Item -Path $M.FullName -Destination $DestinationPath -Force -Recurse
        }
    }
    $ExpandDir | Remove-Item -Force -Recurse
}
```

So what does the script do?

* It verifies if it’s run in at least PowerShell v5. This is required as the Archive CMDlets are available only in V5.
* It takes the path to the resource kit zip file as the Path parameter. The path will be validated and as an additional check, it is verified if the file has the .zip file extension. If the path does not exist or the file does not have the zip file extension, script execution is canceled.
* It provides the invoker with a force switch which will forcefully overwrite all modules with the resource kit content unless the module which is already on the system is newer (handy if you made a manual module change (which is not the best practice by the way! Create a community copy instead) and want to revert back to the original.
* It will create a temporary directory in the Temp folder to expand the resource kit zip file to.
* It will unblock the Zip file (unblocking all Zip content with it).
* It will expand the resource kit zip file to the temporary directory.
* It iterates through every module available in the resource kit and does the following: 
    * It tests if the destination path already exists (which would mean the module is already installed. 
        * If the module already exists, the module on the system and the module from the resource kit are checked for their version. 
            * If the version in the resource kit is newer, the module will be overwritten.
            * If the version in the resource kit is older, a verbose message will always be printed informing the invoker to manually remove the existing module if so desired (fail save).
        * If the Force switch was specified while invoking and the currently installed version is not newer, the module will be overwritten by the module from the resource kit.
    * If the module does not exist on the system yet, it is copied.

Run the script from the console.

![](/images/2015-02/BG_VMRole_DSC_P05P001.png)

When the script is done, you can validate the resources being available by running Get-DSCResource.

# Populating the Pull Server Modules repository

I created a little script to populate the Pull Server Modules repository directory as well.

```powershell
#requires -version 5
param
(
    [Parameter(Mandatory=$true,
                HelpMessage='Enter a filepath to the Resource Kit Zip File.')]
    [ValidateScript({
                        if ((Test-Path -Path $_) -and ($_.split('.')[-1]) -eq 'zip')
                        {
                            $true
                        }
                        else
                        {
                            $false
                        }
                    })]
    [String]$Path,
 
    [Parameter(HelpMessage="Provide the path to the DSC Pull Server Module repository. Defaults to: `$env:ProgramFiles\WindowsPowerShell\DscService\Modules")]
    [String]$PullServerModulesPath = "$env:ProgramFiles\WindowsPowerShell\DscService\Modules"
)
Process
{
    if (Test-Path -Path $PullServerModulesPath)
    {
        $ExpandDir = New-Item -Path $env:TEMP -Name ResourceKitExtract -ItemType Directory -Force
        Unblock-File -Path $Path
        Expand-Archive -Path $Path -DestinationPath $ExpandDir.FullName -Force
        $Modules = Get-ChildItem -Path "$($ExpandDir.FullName)\All Resources"
        foreach ($M in $Modules)
        {
            $ModuleName = $M.Name
            $version = ((Get-ChildItem $M.FullName -Filter '*.psd1' | Get-Content | Select-String "ModuleVersion").tostring()).substring(16).Replace("'", "")
            $ZipName = $ModuleName + "_" + $version + ".zip"
           
            if (!(Test-Path "$PullServerModulesPath\$ZipName"))
            {
                Write-Verbose -Message "Adding $ModuleName with version $version to DSC Pull Server Modules repository" -Verbose
                #clear readonly (compress archive seems to break with access denied if this is set)
                Get-ChildItem $M.FullName -Recurse | %{attrib -R $_.FullName}
                Compress-Archive -Path $M.FullName -DestinationPath "$PullServerModulesPath\$ZipName"
                New-DSCCheckSum -Path $PullServerModulesPath\$ZipName -Force
            }
            else
            {
                Write-Verbose -Message "Module $ModuleName with version $version is already present at DSC Pull Server Modules repository"
            }
 
        }
        $ExpandDir | Remove-Item -Force -Recurse
    }
    else
    {
        Write-Error -Message "Pull Server Module Path not found: $PullServerModulesPath"
    }
}
```

So what does the script do?

* It takes the path to the resource kit zip file as the Path parameter. The path will be validated and as an additional check, it is verified if the file has the .zip file extension. If the path does not exist or the file does not have the zip file extension, script execution is canceled.
* It will accept a non-default path to the DSC Pull Server Modules repository.
* It tests if the DSC Pull Server Modules repository directory exists. When it does not exist script execution is halted.
* It will create a temporary directory in the Temp folder to expand the resource kit zip file to.
* It will unblock the Zip file (unblocking all Zip content with it).
* It will expand the resource kit zip file to the temporary directory.
* It iterates through every module available in the resource kit and does the following: 
    * Gets the module name.
    * Gets the module version number.
    * Constructs the new Module Zip file name (Modules added to the Pull Server repository must be zipped and have a filename in the format: MODULENAME_Version.zip).
    * Tests if the current version of the module is already present in the repository and if it is not 
        * Removes the read-only attribute from all module files (the Compress-Archive CMDlet does not work very well (yet?) if this file attribute is set).
        * Creates the Zip file containing the module content at the DSC Pull Server module repostitory
        * Creates the module checksum file.

Older version should not be removed as configuration documents specify the version they are constructed with explicitly. So if older configurations are still pulled, the module should stay available.

Run the script:

![](/images/2015-02/BG_VMRole_DSC_P05P002.png)

To validate, navigate to the repository directory:

![](/images/2015-02/BG_VMRole_DSC_P05P003.png)

That’s it for this post. Next we will create the PFX website.