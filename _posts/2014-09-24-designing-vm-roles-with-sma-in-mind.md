---
title:  "Designing VM Roles with SMA in mind"
date:   2014-09-24 12:00:00
categories: ["SMA","WAPACK","Windows Azure Pack"]
tags: ["SMA","WAPACK","Windows Azure Pack","PowerShell"]
---

This blog post is written and published on Hyper-V.nu: [https://hyper-v.nu/archives/bgelens/2014/09/designing-vm-roles-with-sma-in-mind/](https://hyper-v.nu/archives/bgelens/2014/09/designing-vm-roles-with-sma-in-mind/){:target="_blank"}

SMA Runbooks can be triggered by Windows Azure Pack events like VM Role creation. You can configure this by going to the VM Cloud automation tab and linking a runbook to an Azure Pack known object like *“Microsoft VMRole“*.

![](/images/2014-09/092314_1923_DesigningWA1.png)

The objects have actionable event like *Create, Scale, Repair and Delete*. You can link one runbook per actionable event per object.

![](/images/2014-09/092314_1923_DesigningWA2.png)

A runbook is only available to linkup with an event if it is configured with the *SPF* tag.

![](/images/2014-09/092314_1923_DesigningWA3.png)

When taking this information in mind, you can see that if you are going to build a dependency between actionable events and SMA runbooks you need to develop and commit to some standards. In this blog post I’ll show you some examples of what you could think off while developing your SMA Windows Azure Pack integration strategy for VM Roles.

# Background information

In my daily job I see VM Roles being used by enterprise customers as a replacement for SCVMM service templates and VM templates. Although the VM Role is meant to be disposable, scalable, recyclable and easily replaceable, they are actually being used for VMs which will have long life spans and take advantage of some of the VM Role benefits like scaling and resource extension configuration.

As you probably know VM Roles make use of differencing disks for both OS disks and Data disks. This is probably not a very good option for VMs which will stay in place for a couple of years. Risks with differencing disks like parent disk corruption become more relevant and have more impact with long term deployments. I’ve seen some comments in the Azure Pack feedback forums asking for VM Roles to be associated with standard VHDX files which received a lot of support. I hope this request could make it to be a vNext feature. Until then you could use SMA runbooks to convert the differencing OS disk to a dynamically expanding “autonomous” disk like the runbook published by [Daniel Neumann](https://twitter.com/neumanndaniel){:target="_blank"} which can be found [here](http://gallery.technet.microsoft.com/SMA-Runbook-Convert-WAP-VM-148937ee){:target="_blank"}.

For test environments / deployments, the use of differencing disks is probably ok and thus you would probably want to trigger a conversion of the disk only if the users input or parameter value justifies it. Maybe you even have some VM Roles planned which should not be triggering SMA runbooks at all. 

# Solution

As you cannot filter your actionable events before passing them on to SMA, you have to take care of filtering after the event has triggered an SMA runbook. So this means you will have to build a master runbook which deals with the filtering. The SMA runbook needs to have a way to find out if SMA processing is desired and if so, what actions to trigger for the VM Role.

A WAPack triggered SMA runbook will have a few objects passed to it by SPF:

* *Params* – contains the parameters defined in the resdefpkg of the VM Role which are exposed to the tenant user via the view definition (the tenant user configurable settings).
* *ResourceObject* – contains all parameter information, the information needed by the resource extension and much more.
* *VMMJobID* – the SCVMM job GUID

There is also the *PSPrivateMetaData* variable which specifies a hash table object that contains user and application level information. It is not SMA or WAPack specific but a standard workflow variable which could be parsed for the information needed. In this blog post it will not be used. I only mention it so you know it exists.

It is not possible to expand the resource definition with name value pairs which are outside of the scope of what is expected by WAPack or SPF. It is possible to add parameters but you cannot hardcode the value. You have to use the view definition to fill these values in. So when expanding the VM Role with the data we want to be picked up by SMA, we will create parameters and add them to the view definition. Then we fill in the default values and make the input fields read–only. These fields are unfortunately displayed to the tenant user. To make this less impactful we will make a new section and position on the last page. A section will be configured which states *“the settings presented on this page can be ignored“*. Fields which rely on the user input will be placed on another section of course.

## Strategy

It is easy to see that if you are going to write a runbook for filtering you should use parameter names which are consistent for all VM Roles so you know what to expect when the object is passed. The strategy is to create a central place where all SMA related parameters are registered and make this document a mandatory read through when VM Roles are designed.

Secondly, a mandatory SMA opt-in parameter will be part of every VM Role. Depending on the value, the SMA runbook decides if it should start child runbooks or not.

## Prerequisites

In this example we are going to download the *“Windows Server 2012 R2 joined to Workgroup Gallery Resource”* and adjust it for SMA processing. You can download the resource using the Web Platform Installer and adding the Service Models feed. For more information on how this works see this [blog](http://blogs.technet.com/b/scvmm/archive/2013/11/04/using-the-service-models-web-platform-installer-gallery.aspx){:target="_blank"}. Go through the readme and make sure you have met all prerequisites for this VM Role (VHD tagging!). We use this resource because it’s an easy starting point.

![](/images/2014-09/092314_1923_DesigningWA4.png)

Once the resource is downloaded, we start modification of it in the VM Role Authoring Tool which can be downloaded here: [https://vmroleauthor.codeplex.com/](https://vmroleauthor.codeplex.com/){:target="_blank"}

We will dive right in, if you need a deep dive on VM Roles, you should definitely first read [Marc van Eijk’s](https://twitter.com/_marcvaneijk){:target="_blank"} blog series starting [here](http://blogs.technet.com/b/privatecloud/archive/2014/07/17/the-windows-azure-pack-vm-role-introduction.aspx){:target="_blank"}.

# Modifying the resource definition – Enable for SMA Processing (Opt-in)

Once you have opened the *VM Role resdefpkg* file in the VM Role authoring tool you should first navigate to the storage profile node and remove the data disk. This will make deployment of this VM role less dependent on resources available in the VMM library.

![](/images/2014-09/092314_1923_DesigningWA5.png)

We will remove all language resources but *“en-US“*. When this is done, we will remove all parametername values involved with the datadisk (the remove button is at the bottom of the page!).

![](/images/2014-09/092314_1923_DesigningWA6.png)

Next we navigate to the parameters node and add a parameter called *“SMAEnabled“*. Of course this parameter is then documented so it will be named the same for every VM Role created hereafter.

![](/images/2014-09/092314_1923_DesigningWA7.png)

The variable will be configured as a string. It would probably make more sense to make it a Boolean but there is a problem in the view definition concerning Booleans which does not allow them to be presented as read-only (bug).

Next we will navigate to the view definition node and we will add a new view definition section for the SMA parameters.

![](/images/2014-09/092314_1923_DesigningWA8.png)

We will give it a name like *“SMA Settings”* and add a view definition category to it (it would be a good strategy to also make the Section and Category names part of the VM Role standard for a consistent experience).

![](/images/2014-09/092314_1923_DesigningWA9.png)

We will call the category: *“The settings presented on this page can be ignored.”*

![](/images/2014-09/092314_1923_DesigningWA10.png)

Next we will add the *“SMAEnabled”* Parameter.

![](/images/2014-09/092314_1923_DesigningWA11.png)

We will configure the parameter with a default value and make it be presented as a read-only string.

![](/images/2014-09/092314_1923_DesigningWA12.png)

Next we will validate the package and we will be confronted with an error about the SMAEnabled parameter. This can be safely ignored as it does not stop us from importing it into the gallery.

![](/images/2014-09/092314_1923_DesigningWA13.png)

Save the resource definition package and import it into the gallery. Make it public and assign it to a plan to make it available.

![](/images/2014-09/092314_1923_DesigningWA14.png)

Log in to the tenant portal and check if the VM Role has been made available.

![](/images/2014-09/092314_1923_DesigningWA15.png)

Run a test deployment to make sure everything is fine. Pay attention at the last page of the wizard to see if settings have been applied correctly (e.g. read–only).

![](/images/2014-09/092314_1923_DesigningWA16.png)

You now have successfully extended the resource definition with a parameter and value which can be used by SMA. You can of course add additional parameters and values besides this *SMAEnabled* parameter. You could for example create a parameter to specify which childrunbook to run or to code a value needed by a parameter of a childrunbook. Whatever you need.

## Creating and linking the filtering runbook

Now we will add a runbook to SMA and link it up to the *“VM Role create event“*.

First we will create the runbook called *VMRole_Create_SMA_Trigger* and add the *SPF* tag.

![](/images/2014-09/092314_1923_DesigningWA17.png)

Then we add the following code to the runbook and publish it.

```powershell
workflow VMRole_Create_SMA_Trigger
{
    param
    (
        [object]$resourceObject,
        [object]$params,
        [string]$vmmjobid
    )
    $toexpend = $resourceObject.ResourceConfiguration
    $toexpend2 = $toexpend.ParameterValues
    $toexpend3 = $toexpend2 -split (',') -replace ('"','') -replace ('{','') -replace ('}','')
    $usableobject = inlinescript {
        $paramhashtable = @{}
        Foreach ($p in $using:toexpend3)
        {
                $value = ""
                $key = ($p -split (':'))[0]
                if ($p -split (':').count -gt 2) 
                {
                    $p -split (':') | ?{$_ -ne $key} | %{
                        $value += $_ + ":"
                    }
                    $value = $value.trimend(":")
                }
                else
                {
                    $value = ($p -split (':'))[1]
                }
                $paramhashtable.Add($key,$value)
        }
        return $paramhashtable
    }
    $usableobject
    if ($usableobject.SMAEnabled -eq "Enabled")
    {
        Write-Output "SMA Processing desired"
    }
    else
    {
        Write-Output "No SMA Processing desired"
    }
}
```

The code takes all objects which are passed by *SPF* as parameter. Then it parses the data in the resourceobject *resourceconfiguration* attribute into a usable hashtable. I based this on a post by [Michael Rueefli](http://www.twitter.com/drmiru){:target="_blank"} which can be found [here](http://www.miru.ch/linking-sma-runbooks-to-azure-pack-vm-cloud-events-and-get-job-parameters){:target="_blank"}.

Note that I slightly adjusted it to take the possibility of multiple values per key into account (you will see this in the screenshot of the runbooks output later on) and to better work against the resourceobject object.

Next we linkup the runbook to the the *MicrosoftCompute VMRole Create action*.

![](/images/2014-09/092314_1923_DesigningWA18.png)

![](/images/2014-09/092314_1923_DesigningWA19.png)

Now let’s deploy the VM Role using a tenant user and watch what the runbook will output.

![](/images/2014-09/092314_1923_DesigningWA20.png)

As you can see, the runbook successfully identified if SMA processing would be desired. We now have a base runbook from which we can expand on. Note that some keys have multiple values as I mentioned before. Multiple values are separated by “:”.

# Modifying the resource definition – User input

Now we modify the VM Role resource definition with a selection field for the user. The user will define if a VM will be expected to have a long or short lifecycle. Based on the user choice we will convert the differencing OS disk to a normal disk using Daniel Neumann SMA runbook or not.

First we increment the resource definition and view definition version to 1.0.0.1.

![](/images/2014-09/092314_1923_DesigningWA21.png)

Then we will add a string parameter called *VMRoleLifecycle*.

![](/images/2014-09/092314_1923_DesigningWA22.png)

We create a new view definition category at the *WS2012R2VMSettings* section called *“VM Lifecycle”* and add the VMRoleLifecycle parameter to it.

![](/images/2014-09/092314_1923_DesigningWA23.png)

We will configure the type as an option and add 2 keys: *“Short”* and *“Long”* which we will give user friendly descriptions of less then or more than 6 months respectively. A default must be configured because when you don’t do this and the user does not interact with the drop down menu, no key is selected.

After validation (ignore unused parameter errors) we will save the resource definition package, import it into the gallery, make it public and assign it to a plan. Perform a test deployment and verify the VM is deployed and what the output is of the *“VMRole_Create_SMA_Trigger”* runbook. 

![](/images/2014-09/092314_1923_DesigningWA24.png)

![](/images/2014-09/092314_1923_DesigningWA25.png)

![](/images/2014-09/092314_1923_DesigningWA26.png)

As you can see, the *VMRoleLifecycle* is passed as expected.

# Implementing convert OS disk runbook as a child runbook

First we create a runbook based on the one published by Daniel Neumann but better fitting for this post (extra param block and less code). Make sure you have the required assets implemented (string variable called VMMServer and credential called SCVMM Service account).

```powershell
workflow Invoke-SCVHDDiffConvert
{
param(
    [string]$vmmjobid
)
    $vmm = Get-AutomationVariable -Name 'VMMServer'
    $vmmcred = Get-AutomationPSCredential -Name 'SCVMM Service Account'


    $Result=InlineScript{
        While((Get-SCJob -ID $USING:VMMJobId).Status -eq 'Running')
        {
            $Progress = (Get-SCJob -ID $USING:VMMJobId).ProgressValue
            Start-Sleep -Seconds 3
        }

        $Result = (Get-SCJob -ID $USING:VMMJobId).Status
        Return $Result
    } -PSComputer $VMM -PSCredential $vmmcred -PSRequiredModule virtualmachinemanager

    if($Result.Value -eq 'Completed'){
        $VMName=InlineScript{
        $SCJob=Get-SCJob -ID $USING:VMMJobId
        $CloudResources= (Get-CloudResource -Id $SCJob.ResultObjectID).vms.name

        foreach($VM in $CloudResources)
        {
            $VM=Get-SCVirtualMachine -Name $VM
            $VMStatus=$VM.Status
            #Shutdown-VM
            if($VMStatus -eq "Running")
            {
                $StopVM=Stop-SCVirtualMachine -VM $VM -Shutdown
            }
            $Disk=$VM.VirtualDiskDrives|Where-Object {$_.BusType -eq "IDE" -and $_.Bus -eq 0 -and $_.Lun -eq 0}
            $VHD=$VM.VirtualHardDisks|Where-Object {$_.ID -eq $Disk.VirtualHardDiskId}
            if($VHD.VHDType -eq "Differencing")
            {
                $SCJob=New-SCExternalJob -Name 'Converting differencing virtual hard disk' -ResultObject $VM
                $SCJobUpdate=Set-SCExternalJob -Job $SCJob -ProgressValue 50
                $UpdateVM=Read-SCVirtualMachine -VM $VM
                Invoke-Command -ComputerName $VM.HostName -Credential $Using:vmmcred -ScriptBlock {
                    param($VM,$VHD)
                    if((Test-Path -Path $VHD.Location) -and $VHD.Location -like "*.vhdx")
                    {
                        $NewName=$VM.Name+".vhdx"
                        $NewPath=$VHD.Directory+"\"+$NewName
                        $Convert=Convert-VHD -Path $VHD.Location -DestinationPath $NewPath -VHDType Dynamic -DeleteSource
                        $Set=Set-VMHardDiskDrive -ComputerName $VM.HostName  -VMName $VM.Name -ControllerType IDE -Path $NewPath -ControllerNumber 0 -ControllerLocation 0 -AllowUnverifiedPaths -ErrorAction SilentlyContinue
                    }
                    if((Test-Path -Path $VHD.Location) -and $VHD.Location -like "*.vhd")
                    {
                        $NewName=$VM.Name+".vhd"
                        $NewPath=$VHD.Directory+"\"+$NewName
                        $Convert=Convert-VHD -Path $VHD.Location -DestinationPath $NewPath -VHDType Dynamic -DeleteSource
                        $Set=Set-VMHardDiskDrive -ComputerName $VM.HostName  -VMName $VM.Name -ControllerType IDE -Path $NewPath -ControllerNumber 0 -ControllerLocation 0 -AllowUnverifiedPaths -ErrorAction SilentlyContinue
                    }
                } -ArgumentList $VM,$VHD
                $SCJobUpdate=Set-SCExternalJob -Job $SCJob -Completed
                #VM Refresh
                $UpdateVM=Read-SCVirtualMachine -VM $VM
                $StartVM=Start-SCVirtualMachine -VM $VM                
            }
        }
    } -PSComputer $VMM -PSCredential $vmmcred -PSRequiredModule virtualmachinemanager
    }
    else
    {
        write-error "VM role not succesfully deployed, ignoring disk conversion"
    }               
}
```

Then we expand the *VMRole_Create_SMA_Trigger* runbook so it will call the *Invoke-SCVHDDiffConvert* runbook when SMA processing is desired and the VM Role Lifecycle is expected to be long. We will also implement an SMA string variable asset called *SMAWebEndpoint* which should hold the URL of the SMA web service (e.g. https://sma.domain.local). It is assumed that the runbook worker account is an SMA administrator for this runbook to work. When this is not desired, an extra SMA credential asset should be created which can be passed to the *Start-SmaRunbook* cmdlet.

```powershell
workflow VMRole_Create_SMA_Trigger
{
    param
    (
        [object]$resourceObject,
        [object]$params,
        [string]$vmmjobid
    )
    $smawebserver = Get-AutomationVariable -Name 'SMAWebEndpoint'
    $smawebserverport = "9090"

    $toexpend = $resourceObject.ResourceConfiguration
    $toexpend2 = $toexpend.ParameterValues
    $toexpend3 = $toexpend2 -split (',') -replace ('"','') -replace ('{','') -replace ('}','')
    $usableobject = inlinescript {
        $paramhashtable = @{}
        Foreach ($p in $using:toexpend3)
        {
                $value = ""
                $key = ($p -split (':'))[0]
                if ($p -split (':').count -gt 2) 
                {
                    $p -split (':') | ?{$_ -ne $key} | %{
                        $value += $_ + ":"
                    }
                    $value = $value.trimend(":")
                }
                else
                {
                    $value = ($p -split (':'))[1]
                }
                $paramhashtable.Add($key,$value)
        }
        return $paramhashtable
    }
    $usableobject
    if ($usableobject.SMAEnabled -eq "Enabled")
    {
        #check if VM role will have a long or short lifecycle
        if ($usableobject.VMRoleLifecycle -eq "Long")
        {
            #start OS disk conversion
            $job = Start-SmaRunbook -WebServiceEndpoint $smawebserver -Port $smawebserverport -Name "Invoke-SCVHDDiffConvert" -Parameters  @{"vmmjobid"=$vmmjobid}
            do
            {
                $jobinfo = Get-SmaJob -WebServiceEndpoint $smawebserver -Port $smawebserverport -Id $job
            }
            while ($jobinfo.jobstatus -ne "Completed")
        }
        else
        {
            #no childrunbook yet for short lifecycle
        }
    }
    else
    {
        Write-Output "No SMA Processing desired"
    }
}
```

When a tenant users deploys the VM Role version 1.0.0.1 and select the lifecycle of more than 6 months, the VHD(X) is converted.

# Conclusion

Although you can only have 1 runbook per actionable event per object, when planned and documented in the right way you can build yourself a solid framework which is highly expandable.

The tip of this blog is to look ahead of what you need at this moment as if you don’t, it can put your WAPack automation at risk in the future.

# Bonus: Alternative solution

After I send in this blog post to Hans and Marc for review I was soon contacted. Both were very enthusiastic about the work I had already done. They wanted to look if we could expand this post with an alternative solution. The solution should not bother the tenant user with the view definition expansion which holds the data for the SMAEnabled parameter. As I said earlier, it is not possible to expand the resource definition outside of the scope what is expected by WAPack and SPF. We discussed about a lot of possibilities and eventually we found one which should work. I started working on the scenario in my lab and this resulted in this bonus section!

The strategy for this alternative solution is to modify the VM Role resource definition name with a postfix “–SMA“. The idea is, if the SMA post fix is present, SMA processing should occur.

![](/images/2014-09/092314_1923_DesigningWA27.png)

A caveat with this option is that it will branch off another VM Role to the tenant user.

![](/images/2014-09/092314_1923_DesigningWA28.png)

The tenant user sees the label defined in the view definition. When using this method, you should change the label to differentiate between a non-SMA and SMA enabled runbook (e.g. *“Windows Server 2012 R2 – Basic”* and *“Windows Server 2012 R2 – Premium“*).

When using this method the alternative runbook would look something like this:

```powershell
workflow VMRole_Create_SMA_Trigger2
{
    param
    (
        [object]$resourceObject,
        [object]$params,
        [string]$vmmjobid
    )
    #Change trigger filter to check for SMA post fix
    if (($resourceObject.ResourceDefinition.name -split ('-') | select -last 1) -eq "SMA")
    {
        Write-Output "SMA Processing desired"
    }
    else
    {
        Write-Output "No SMA Processing desired"
    }
} 
```

And as you can see from the output, this works equally well.

![](/images/2014-09/092314_1923_DesigningWA29.png)

Whichever solution fits you best, the same principles apply. You should think about this up front and develop a strategy. Either by a naming convention or by expanding the parameters.

HTH

Ben Gelens