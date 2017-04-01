---
title:  "Integrating VM Role with Desired State Configuration Part 9 – Create a Domain Join DSC resource"
date:   2015-02-23 12:00:00
categories: ["WAPACK","Windows Azure Pack","Desired State Configuration","PSDSC"]
tags: ["WAPACK","Windows Azure Pack","PowerShell","VM Role","PKI","PSDSC","Desired State Configuration"]
---

This blog post is written and published on Hyper-V.nu: [http://hyper-v.nu/archives/bgelens/2015/02/integrating-vm-role-with-desired-state-configuration-part-9-create-a-domain-join-dsc-resource/](http://hyper-v.nu/archives/bgelens/2015/02/integrating-vm-role-with-desired-state-configuration-part-9-create-a-domain-join-dsc-resource/){:target="_blank"}

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

In a previous post I talked about why I did not include a domain join in my example DSC configuration:
>So why not incorporate a domain join in this configuration? There is a resource available in the Resource Kit which can handle this right?
>Yes, there is a resource for this and a domain join would be the most practical example I would come up with as well. But….
>The xComputer DSC resource contained in the xComputerManagement module has a mandatory parameter for the ComputerName. As I don’t know the ComputerName up front (the ComputerName is assigned by VMM based on the name range provided in the resource definition), I cannot generate a configuration file up front. I could deploy a VM Role with just 1 instance containing a ComputerName which was pre-defined and used in a configuration document but this scenario is very inflexible and undesirable. In a later post in this series I will show you how to create a DSC Resource yourself to handle the domain join without the ComputerName to be known up front.

In this blog post we will author a DSC resource which handles domain joining without the need to know the ComputerName up front which makes it a suitable resource for the Pull Server scenario described in this series.

# WMF 5 Preview release

When I started writing this blog series, Windows Management Foundation 5 (WMF 5) November Preview was the latest WMF 5 installment available for Windows Server 2012 R2.

I created a DSC resource to handle the domain joining using the new Class based definition method which was at the time defined as “experimental design” (see [http://blogs.msdn.com/b/powershell/archive/2014/12/10/wmf-5-0-preview-defining-quot-experimental-designs-quot-and-quot-stable-designs-quot.aspx](http://blogs.msdn.com/b/powershell/archive/2014/12/10/wmf-5-0-preview-defining-quot-experimental-designs-quot-and-quot-stable-designs-quot.aspx){:target="_blank"} for more info). I did this because this is the way forward for authoring DSC resources and I find the experience building the resource to be a lot easier in this fashion.

Then WMF 5 February Preview came along and broke my resource by changing the parameter attributes (e.g. [DscResourceKey()] became [DscProperty(Key)] and [DscResourceMandatory()] became [DscProperty(Mandatory)]). I fixed the resource for the February Preview release (download WMF 5 Feb preview here: [http://www.microsoft.com/en-us/download/details.aspx?id=45883](http://www.microsoft.com/en-us/download/details.aspx?id=45883){:target="_blank"}).

The DSC resource Class definition is now declared as a “stable design”. Because of this I don’t expect much changes anymore and if a change would be made, repairing the resource should be relatively easy.

***

If you want more insights on why it is easier using class defined resources opposed to the WMF 4 introduced ‘script module’ resources, I advise you to look at an excellent blog post created by Ravikanth Chaganti here: [http://www.powershellmagazine.com/2014/10/06/class-defined-dsc-resources-in-windows-management-framework-5-0-preview/](http://www.powershellmagazine.com/2014/10/06/class-defined-dsc-resources-in-windows-management-framework-5-0-preview/){:target="_blank"}. If you want to know more about the PowerShell Class implementation in general you should definitely have a Beer with Trevor Sullivan here: [http://trevorsullivan.net/2014/10/25/implementing-a-net-class-in-powershell-v5/](http://trevorsullivan.net/2014/10/25/implementing-a-net-class-in-powershell-v5/){:target="_blank"}.

***

I tested my resource locally (by adding it to the module directory directly) and it worked great. I though I had done it, a Pull server scenario friendly resource to handle the domain join without the need to provide the computer name up front using the latest and greatest Class based syntax.

So I prepped the resource to be deployed for the first time via the Pull server and I was hit by a problem. I expected for modules with Class defined resources to just work when delivered via the Pull server. The Pull server itself actually has no problems with them at all but the Web Download Manager component of the LCM is apparently hard wired to check for a valid “script module” structure (at the time of writing using the February Preview).

![](/images/2015-02/BG_VMRole_DSC_P09P001.png)

![](/images/2015-02/BG_VMRole_DSC_P09P002.png)

As a workaround, you could add the Class defined module to the “C:\Program Files\WindowsPowerShell\Modules” directory of your VM Role images directly. This will result in the download of the module to be skipped as it is already present (but you actually don’t want to do this because it is undesirable to maintain DSC resources in an OS image).

![](/images/2015-02/BG_VMRole_DSC_P09P003.png)

Because WMF 5 is not RTM yet, I think the behavior of the WebDownloadManager will be adjusted in a later version. In the mean time I have logged this behavior on connect here: [https://connect.microsoft.com/PowerShell/feedback/details/1143212](https://connect.microsoft.com/PowerShell/feedback/details/1143212){:target="_blank"}.

To make this post a bit more future proof, I will show you how to author both the Class based module and the script based module. Although you can only use the script based module today, the Class based module should be usable in the near future as well.

# Author Domain Join DSC Resource Class Resource

A DSC Resource is part of a DSC module. You can have just one resource in your module or you could have multiple resources in your module. In this case we will create a DSC module called cMyDSCModule and will add a resource cDomainJoin.

First we create a directory and name it cMyDSCModule then we add a PowerShell module file (.PSM1) to it which is also named cMyDSCModule.

![](/images/2015-02/BG_VMRole_DSC_P09P004.png)

Copy and save the following content in the cMyDSCModule.psm1 module file:

```powershell
enum Ensure
{
   Absent
   Present
}
 
[DscResource()]
class cDomainJoin
{
 
    [DscProperty(Mandatory)]
    [Ensure]$Ensure
 
    [DscProperty(Key)]
    [String]$Domain
 
    [DscProperty(Mandatory)]
    [PSCredential]$Credential
 
    [Void]Set()
    {
        if ($this.Ensure -eq [Ensure]::Present)
        {
            try
            {
                Add-Computer -DomainName $this.Domain -Credential $this.Credential -Force -PassThru -ErrorAction Stop
                $global:DSCMachineStatus = 1
            }
            catch
            {
                throw 'Exception happened joining computer to domain'
            }
            }
        else
        {
            try
            {
                Remove-Computer -UnjoinDomainCredential $this.Credential -WorkgroupName 'Workgroup' -Force -PassThru -ErrorAction Stop
                $global:DSCMachineStatus = 1
            }
            catch
            {
                throw 'Exception happened removing computer from domain'
            }
        }
    }
    
    [Bool]Test()
    {
        if ($this.Ensure -eq [Ensure]::Present)
        {
            Write-Verbose "Checking if the computer is a member of domain: $($this.Domain)"
            if ((Get-CimInstance -ClassName Win32_ComputerSystem).Domain -eq $this.Domain)
            {
                Write-Verbose "Computer is member of the domain: $($this.Domain)"
                return $true
            }
            else
            {
                Write-Verbose "Computer is not a member of the domain: $($this.Domain)"
                return $false
            }
        }
        else
        {
            Write-Verbose -Message "Checking if the computer is a member of the Workgroup"
            if ((Get-CimInstance -ClassName Win32_ComputerSystem).PartOfDomain)
            {
                Write-Verbose "Computer is member of a domain"
                return $false
            }
            else
            {
                Write-Verbose "Computer is a Workgroup member"
                return $true
            }
        }
    }
 
    [cDomainJoin]Get()
    {
        $Environment = Get-CimInstance -ClassName Win32_ComputerSystem
        $Configuration = [hashtable]::new()
        $Configuration.Add('Domain',$this.Domain)
        $Configuration.Add('Credential',$this.Credential)
        if ($this.Domain -eq $Environment.Domain)
        {
            $Configuration.Add('Ensure','Present')
        }
        else
        {
            $Configuration.Add('Ensure','Absent')
        }
        return $Configuration
    }

}
```

As you can see, a DSC resource class of cDomainJoin is added to the module. The class in this case only has the three mandatory methods implemented (Set, Test and Get), no helper methods are defined.

* The Set method: joins or leaves the defined domain depending if the configuration ensures domain membership to be present or absent.
    Note that the DSCMachineStatus variable is set to 1. This will inform the LCM that a reboot is required.
* The Test method: checks if the desired state is reached or not.
* The Get method: outputs the current state.

Now the module file is created, a module manifest needs to be added so the DSC Module and it’s resources are discoverable and usable.

Adjust the values in the Params hashtable if needed before you run it.

```powershell
$params = @{
    Guid                 = [System.Guid]::NewGuid().guid
    Path                 = 'C:\cMyDSCModule\cMyDSCModule.psd1'
    RootModule           = 'cMyDSCModule.psm1'
    ModuleVersion        = '0.0.0.1'
    Author               = 'Ben Gelens'
    CompanyName          = 'Hyper-V.nu'
    PowerShellVersion    = '5.0'
    DscResourcesToExport = '*'
}
 
New-ModuleManifest @params
```

If you cannot run this command because of an invalid parameter, you are not on the WMF 5 February preview! The DscResourcesToExport parameter is new in this release.

You should now have 2 items in the module folder, the DSC resource module file (PSM1) and a manifest as a PowerShell data file (PSD1).

![](/images/2015-02/BG_VMRole_DSC_P09P005.png)

Next copy the module to a configuration document authoring station (in my case this is the DSC Pull Server) and put it in the directory C:\Program Files\WindowsPowerShell\Modules. 

![](/images/2015-02/BG_VMRole_DSC_P09P006.png)

Now let’s see if the DSC resource module and it’s resource can be discovered.

Run the following command:

```powershell
Get-DscResource -Module cMyDSCModule
```

If you have an error instead of results, you are not on the WMF 5 February preview yet, which is the first time the -Module parameter was added to the Get-DscResource cmdlet.

In earlier WMF 5 builds you had to filter the module on the right side of the pipeline (Get-DscResource |?{$_.Module -like ‘cMyDSCModule’}), this would however leave you without results as Class defined DSC resource modules where not discoverable using this cmdlet before the February preview.

WMF 5 November preview:

![](/images/2015-02/BG_VMRole_DSC_P09P007.png)

WMF 5 February preview:

![](/images/2015-02/BG_VMRole_DSC_P09P008.png)

Next we will package the module into a zip file so it is available via the DSC Pull Server module repository.

```powershell
$source = 'C:\Program Files\WindowsPowerShell\Modules\cMyDSCModule'
$ZipName = 'cMyDSCModule_0.0.0.1.zip'
$PullServerModulesPath = "$env:ProgramFiles\WindowsPowerShell\DscService\Modules"
Compress-Archive -Path $source -DestinationPath "$PullServerModulesPath\$ZipName"

New-DSCCheckSum -Path $PullServerModulesPath\$ZipName -Force
```

![](/images/2015-02/BG_VMRole_DSC_P09P009.png)

As you have read earlier, this module can not be delivered via the Pull server at this time because the LCM WebDownloadManager does not know how to handle Class based modules yet. As soon as the module can be delivered via the Pull server, I’ll update this post to let you know. For now, skip the step of adding the module to the Pull server module repository!

# Script Resource

Authoring a script based resource is a bit more complex then the Class based resource. I’ve provided you with some links before which you could visit to better understand why this is the case. Luckily, a PowerShell module called xDSCResourceDesigner has been made available via the DSC resource kits for some time now which makes this process a lot easier. It takes care of constructing the schema and module structure. So make sure you have this module installed before you continue (you can download it as part of the DSC resource kit or directly here: [https://gallery.technet.microsoft.com/scriptcenter/xDscResourceDesigne-Module-22eddb29](https://gallery.technet.microsoft.com/scriptcenter/xDscResourceDesigne-Module-22eddb29){:target="_blank"}).

I won’t go into too much detail on what the module does in this case as everything is discussed already in the Class based resource paragraph.

Run the following script to create the resource skeleton, schema and manifest:

```powershell
$Path = 'C:'
$ModuleName = 'cMyScriptBasedDSCModule'
$ResourceName = 'cScriptDomainJoin'
$domain = New-xDscResourceProperty -Name 'Domain' -Type String -Attribute Key
$ensure = New-xDscResourceProperty -Name 'Ensure' -Type String -Attribute Required -ValidateSet 'Present','Absent'
$cred = New-xDscResourceProperty -Name 'Credential' -Type PSCredential -Attribute Required
New-xDscResource -Name $ResourceName -ModuleName $ModuleName -Property $domain, $ensure, $cred -Path $Path\
(Get-Content $Path\$ModuleName\$ModuleName.psd1).Replace({ModuleVersion = '1.0'},{ModuleVersion = '0.0.0.1'}) | Out-File $Path\$ModuleName\$ModuleName.psd1
```

The following file and folder structure is created by the New-xDscResource cmdlet.

![](/images/2015-02/BG_VMRole_DSC_P09P018a.png)

To differentiate between the Class based resource and this script based resource, I called the module cMyScriptBasedDSCModule and called the resource cScriptDomainJoin. Also the module is re-versioned to 0.0.0.1.

Open the cScriptDomainJoin.psm1 file in ISE and replace the content with the following:

```powershell
function Get-TargetResource
{
	[CmdletBinding()]
	[OutputType([System.Collections.Hashtable])]
	param
	(
		[parameter(Mandatory = $true)]
		[System.String]
		$Domain,

		[parameter(Mandatory = $true)]
		[ValidateSet("Present","Absent")]
		[System.String]
		$Ensure,

		[parameter(Mandatory = $true)]
		[System.Management.Automation.PSCredential]
		$Credential
	)

        $Environment = Get-CimInstance -ClassName Win32_ComputerSystem
        $Configuration = [hashtable]::new()
        $Configuration.Add('Domain',$Domain)
        $Configuration.Add('Credential',$Credential)
        if ($Domain -eq $Environment.Domain)
        {
            $Configuration.Add('Ensure','Present')
        }
        else
        {
            $Configuration.Add('Ensure','Absent')
        }
        return $Configuration
}

function Set-TargetResource
{
	[CmdletBinding()]
	param
	(
		[parameter(Mandatory = $true)]
		[System.String]
		$Domain,

		[parameter(Mandatory = $true)]
		[ValidateSet("Present","Absent")]
		[System.String]
		$Ensure,

		[parameter(Mandatory = $true)]
		[System.Management.Automation.PSCredential]
		$Credential
	)

    if ($Ensure -eq 'Present')
    {
        try
        {
            Add-Computer -DomainName $Domain -Credential $Credential -Force -PassThru -ErrorAction Stop
            $global:DSCMachineStatus = 1
        }
        catch
        {
            throw 'Exception happened joining computer to domain'
        }
        }
    else
    {
        try
        {
            Remove-Computer -UnjoinDomainCredential $Credential -WorkgroupName 'Workgroup' -Force -PassThru -ErrorAction Stop
            $global:DSCMachineStatus = 1
        }
        catch
        {
            throw 'Exception happened removing computer from domain'
        }
    }
}

function Test-TargetResource
{
	[CmdletBinding()]
	[OutputType([System.Boolean])]
	param
	(
		[parameter(Mandatory = $true)]
		[System.String]
		$Domain,

		[parameter(Mandatory = $true)]
		[ValidateSet("Present","Absent")]
		[System.String]
		$Ensure,

        [parameter(Mandatory = $true)]
		[System.Management.Automation.PSCredential]
		$Credential
	)

    if ($Ensure -eq 'Present')
    {
        Write-Verbose "Checking if the computer is a member of domain: $Domain"
        if ((Get-CimInstance -ClassName Win32_ComputerSystem).Domain -eq $Domain)
        {
            Write-Verbose "Computer is member of the domain: $Domain"
            return $true
        }
        else
        {
            Write-Verbose "Computer is not a member of the domain: $Domain"
            return $false
        }
    }
    else
    {
        Write-Verbose -Message "Checking if the computer is a member of the Workgroup"
        if ((Get-CimInstance -ClassName Win32_ComputerSystem).PartOfDomain)
        {
            Write-Verbose "Computer is member of a domain"
            return $false
        }
        else
        {
            Write-Verbose "Computer is member a Workgroup member"
            return $true
        }
    }
}

Export-ModuleMember -Function *-TargetResource
```

Next copy the module to the configuration authoring station in directory C:\Program Files\WindowsPowerShell\Modules so it is available to write configurations with.

Next the module is packaged into a zip file so it is available via the DSC Pull Server module repository.

```powershell
$source = 'C:\Program Files\WindowsPowerShell\Modules\cMyScriptBasedDSCModule'
$ZipName = 'cMyScriptBasedDSCModule_0.0.0.1.zip'
$PullServerModulesPath = "$env:ProgramFiles\WindowsPowerShell\DscService\Modules"
Compress-Archive -Path $source -DestinationPath "$PullServerModulesPath\$ZipName"

New-DSCCheckSum -Path $PullServerModulesPath\$ZipName -Force
```

# Service the VM Role

We could service the deployed VM Role instance to domain join in 2 methods:

* Update the already assigned configuration document at the Pull Server and let the LCM pull it’s updated configuration.
* Create a new configuration document and update the LCM configuration ID via the tenant portal at the cloud service level.

## Update the configuration document

First I will show how you could adjust the example configuration which was pulled by the VM Role earlier (please note I added both the script based module and Class based module references but have out commented the Class module references).

```powershell
#region variables
$Node = '2a8a4e1e-84a9-446b-ba66-38f85c924d14'
$certfile = "C:\PublicCerts\$Node.cer"
$LocalAdminCred = (Get-Credential -Message 'Enter new local Admin credentials')
$DomainJoinCred = (Get-Credential -Message 'Enter Domain Join credentials')
$MyDomain = 'Gelens.int'
#endregion variables

#region configuration
configuration LocalAdmin
{
    param
    (
        [String]$Node,

        [PSCredential]$LocalAdminCred,

        [PSCredential]$DomainJoinCred,

        [String]$Domain
    )    
    Import-DscResource -ModuleName xCredSSP
    Import-DscResource -ModuleName cMyScriptBasedDSCModule
    #Would be used when Class module can be deployed via Pull Server
    #Import-DscResource -ModuleName cMyDSCModule
    node $Node
    {
        User LocalAdmin
        {
            UserName = $LocalAdminCred.UserName
            Ensure = 'Present'
            Password = $LocalAdminCred
            Description = 'User created by DSC'
            PasswordNeverExpires = $true
            PasswordChangeNotAllowed = $true
        }

        Group Administrators
        {
            GroupName = 'Administrators'
            MembersToInclude = $LocalAdminCred.UserName
            DependsOn = "[User]LocalAdmin"
        }
        
        xCredSSP CredSSPServer
        {
            Ensure = 'Present'
            Role = 'Server'
        }

        cScriptDomainJoin MyDomain
        {
            Ensure = 'Present'
            Domain = $Domain
            Credential = $DomainJoinCred
        }

        <#
        #Would be used when Class module can be deployed via Pull Server
        cDomainJoin MyDomain
        {
            Ensure = 'Present'
            Domain = $Domain
            Credential = $DomainJoinCred
        }
        #>
    }
}
#endregion configuration

#region configuration data
$ConfigData = @{   
    AllNodes = @(        
        @{     
            NodeName = $Node
            CertificateFile=$certfile
        } 
    )  
} 
#endregion configuration data

#region logic
LocalAdmin -Node $Node `
           -OutputPath 'C:\Program Files\WindowsPowerShell\DscService\Configuration' `
           -ConfigurationData $ConfigData `
           -LocalAdminCred $LocalAdminCred `
           -DomainJoinCred $DomainJoinCred `
           -Domain $MyDomain

New-DSCCheckSum -ConfigurationPath "C:\Program Files\WindowsPowerShell\DscService\Configuration\$Node.mof" -Force
#endregion logic
```

So what changed?

* the script now asks for 2 credentials. The original for the local administrator and a new one for the domain join.
* A domain name has to be provided
* The DSC configuration is now updated with a domain, local admin credential and domain join credential parameter.
* The configurations Import-DscResource now imports 2 modules. The original xCredSSP and the new cMyDSCModule.
* A configuration for the domain join has now been declared.
* The DSC configuration is called providing the extra information to it and the newly generated configuration document (MOF file) is saved to the pull server configuration repository (with all sensitive data encrypted).
* A new DSC checksum is created so the LCM will see it needs to pull and apply the new configuration.

When you have added the updated configuration document to the configuration repository of the Pull server, you just have to wait for a while for the new configuration to be pulled and applied by the LCM. If you are not patient you could force the LCM to pull the new configuration immediately by running Update-DscConfiguration (could be run remotely as well using a CIM session) and force the consistency engine to run immediately after the download by running:

```powershell
Invoke-CimMethod -Namespace root/Microsoft/Windows/DesiredStateConfiguration `
                 –ClassName MSFT_DSCLocalConfigurationManager `
                 -MethodName PerformRequiredConfigurationChecks `
                 -Arg @{Flags = [System.UInt32]1} -Verbose
```

The beauty of this method is that as soon as the VM Role is deployed, DSC takes over and governs the VM’s configuration for the rest of its lifecycle.

## Create a new configuration document

Creating a new configuration document with a new node configuration id allows you to service the VM Role from the tenant portal by modifying the cloud service properties. This would serve best when DSC is used for initial / add-on configuration only and there is no need to monitor for configuration drift and check for constancy. Beware that when you assign a new configuration id, the previous configuration is overwritten by the new configuration document and it therefore it cannot be monitored anymore (also the changes implemented by the previous configuration cannot be resolved by Get-DscConfiguration anymore).

Let’s create a new configuration and see how the servicing is handled through the tenant portal.

First let’s generate a new GUID for a configuration id and request a new certificate for encryption purposes (I can just run this script and be done with it as I write my configurations and store my PFX files on the Pull Server. If you have distributed your roles you would probably need to adjust the script to handle some file copying).

```powershell
#region variables
$GUID = [System.Guid]::NewGuid().guid
#GUID = '44d217e8-c0c3-45b4-ada5-3d9e2a8955f4'
$WebEnrollURL = 'https://webenroll.hyperv.nu/ADPolicyProvider_CEP_UsernamePassword/service.svc/CEP'
$WebEnrollCred = Get-Credential -Message 'Enter Credentials valid for certificate requests'
$Template = 'DSCEncryption'
$PFXPath = 'C:\PFXSite'
$CERPath = 'C:\PublicCerts'
$PFXPwd = ([char[]](Get-Random -Input $(48..57 + 65..90 + 97..122) -Count 12)) -join ""
$SecPFXPwd = $PFXPwd | ConvertTo-SecureString -AsPlainText -Force
#endregion variables
 
#region logic
try
{
    Write-Verbose -Message "Requesting certificate from template: $Template at URI: $WebEnrollURL" -Verbose
    $cert = Get-Certificate -Url $WebEnrollURL -Template $Template -SubjectName "CN=$GUID" -CertStoreLocation Cert:\LocalMachine\My -Credential $WebEnrollCred -ErrorAction Stop
    Write-Verbose -Message "Succesfully requested certificate"
}
catch
{
    throw "Certificate Request failed"
}
Write-Verbose -Message "Exporting certificate with Private and Public Key to PFX at path: $PFXPath" -Verbose
Export-PfxCertificate -Cert $cert.Certificate.PSPath -Password $SecPFXPwd -FilePath "$PFXPath\$($cert.Certificate.Subject.TrimStart('CN=')).pfx" -ChainOption EndEntityCertOnly -Force | Out-Null
$PFXPwd | Out-File -FilePath "$PFXPath\$($cert.Certificate.Subject.TrimStart('CN=')).txt"
 
Write-Verbose -Message "Exporting Certificate with Public key to cer file at path: $CERPath" -Verbose
Export-Certificate -Cert $cert.Certificate.PSPath -FilePath "$CERPath\$($cert.Certificate.Subject.TrimStart('CN=')).cer" -Type CERT -Force | Out-Null
 
Write-Verbose -Message 'Removing certificate from computer store' -Verbose
Remove-Item $cert.Certificate.PSPath -Force
#endregion logic
 
#region output
$Props = @{
    GUID = $GUID
    PWD = $PFXPwd
    PFX = "$PFXPath\$($cert.Certificate.Subject.TrimStart('CN=')).pfx"
    CER = "$CERPath\$($cert.Certificate.Subject.TrimStart('CN=')).cer"
}
New-Object -TypeName PSObject -Property $Props | Format-List
#endregion output
```

![](/images/2015-02/BG_VMRole_DSC_P09P010.png)

We now have a new configuration id (3a595498-8343-4df0-abc1-5f9dd73dc6bc) which we are going to use when generating the configuration (Again, I’ve added the Class based module references but out commented them).

```powershell
#region variables
$Node = '3a595498-8343-4df0-abc1-5f9dd73dc6bc'
$certfile = "C:\PublicCerts\$Node.cer"
$DomainJoinCred = (Get-Credential -Message 'Enter Domain Join credentials')
$MyDomain = 'gelens.int'
#endregion variables
 
#region configuration
configuration DomainJoin
{
    param
    (
        [PSCredential]$DomainJoinCred,

        [String]$Domain
    )    
    Import-DscResource -ModuleName cMyScriptBasedDSCModule

    #Would be used when Class module can be deployed via Pull Server
    #Import-DscResource -ModuleName cMyDSCModule
    node $Node
    {
        cScriptDomainJoin MyDomain
        {
            Ensure = 'Present'
            Domain = $Domain
            Credential = $DomainJoinCred
        }
        <#
        #Would be used when Class module can be deployed via Pull Server
        cDomainJoin MyDomain
        {
            Ensure = 'Present'
            Domain = $Domain
            Credential = $DomainJoinCred
        }
        #>
    }
}
#endregion configuration
 
#region configuration data
$ConfigData = @{   
    AllNodes = @(        
        @{     
            NodeName = $Node
            CertificateFile=$certfile
        } 
    )  
} 
#endregion configuration data
 
#region logic
DomainJoin -Node $Node `
           -OutputPath 'C:\Program Files\WindowsPowerShell\DscService\Configuration' `
           -ConfigurationData $ConfigData `
           -DomainJoinCred $DomainJoinCred `
           -Domain $MyDomain
 
New-DSCCheckSum -ConfigurationPath "C:\Program Files\WindowsPowerShell\DscService\Configuration\$Node.mof" -Force
#endregion logic
```

Running the script results in two files in the Pull Server configuration repository:

![](/images/2015-02/BG_VMRole_DSC_P09P011.png)

Now everything is in place to start servicing the VM Role. Navigate to the Azure Pack tenant portal and login.

![](/images/2015-02/BG_VMRole_DSC_P09P012.png)

Select the Virtual Machines resource provider and select the Cloud Service deployed earlier. Drill down by select the name.

![](/images/2015-02/BG_VMRole_DSC_P09P013.png)

Select the configure Tab.

![](/images/2015-02/BG_VMRole_DSC_P09P014.png)

Change the configuration ID to the new one and press save. A warning will pop-up, select yes to continue with the servicing.

![](/images/2015-02/BG_VMRole_DSC_P09P015.png)

The VM Role is now serviced with the new configuration id. The new PFX is downloaded, the LCM is reconfigured with the new configuration id and encryption certificate thumbprint. Update-DscConfiguration is triggered which will trigger the LCM to download the new configuration document and apply it. When configuration is done, the node is rebooted and you can login using your domain credentials.

![](/images/2015-02/BG_VMRole_DSC_P09P016.png)

Run Test-DscConfiguration, Get-DscConfiguration and Get-DscConfigurationStatus to check results.

![](/images/2015-02/BG_VMRole_DSC_P09P017a.png)

That’s it for this post, next we will finish the series with some closing notes.

Download both the Class and Script based modules of the resource here: [https://1drv.ms/f/s!AjCqd85FSgV_n8cnxleCODNWXR335A](https://1drv.ms/f/s!AjCqd85FSgV_n8cnxleCODNWXR335A){:target="_blank"}