---
title:  "Offline Updating Hyper-V VMs"
date:   2013-09-30 12:00:00
categories: ["Hyper-V","Windows Server"]
tags: ["Hyper-V","Windows Server","PowerShell"]
---
Have a lot of VMs which are offline and want to check if their Integration Tools are up 2 date?
I wrote a little script which does just that.
And as a bonus, when the VM has 2008R2 or higher installed, the Integration Tools are automatically updated with the latest version available on the Hyper-V host you run this on!

HTH

```powershell
$integrationServicesCabPath="C:\Windows\vmguest\support\amd64\Windows6.2-HyperVIntegrationServices-x64.cab"
$VM = get-vm |?{$_.state -eq "off"}
$latestversion2008up = "6.2.9200.16433"
$latestversion2003 = "6.2.9200.16384"

foreach ($v in $vm){
    $vhd = ($v | Get-VMHardDiskDrive -ControllerType IDE).path

    $diskNo=(Mount-VHD -Path $vhd –Passthru).DiskNumber
    $driveLetter=(Get-Disk $diskNo | Get-Partition).DriveLetter
    if ((Get-Disk $diskNo).OperationalStatus -ne 'Online')
    {
        Set-Disk $diskNo -IsOffline:$false -IsReadOnly:$false
    }
    [string]$name = $vm.name
    $serverprops = New-Object System.Collections.Specialized.OrderedDictionary
    $serverprops.add('VMName',$name)

    $osversion = switch -wildcard ((get-item ($driveLetter + ":\windows\system32\ntdll.dll")).VersionInfo.ProductVersion)
    {
        "5.2*" {"Windows Server 2003"}
        "6.0*" {"Windows Server 2008"}
        "6.1*" {"Windows Server 2008R2"}
        "6.2*" {"Windows Server 2012"}
    }
    $serverprops.add('OSVersion',$osversion)

    if ($osversion -eq "Windows Server 2003")
    {
        $serverprops.add('Update',"NA")
        if ((get-item ($driveletter + ":\windows\system32\drivers\vmbus.sys")).VersionInfo.ProductVersion -eq $latestversion2003)
        {
            $serverprops.add('IntegrationTools',"Up2Date")
        }
        else
        {
            $serverprops.add('IntegrationTools',"Outdated")
        }
    }
    elseif ($osversion -eq "Windows Server 2008")
    {
        $serverprops.add('Update',"NA")
        if ((get-item ($driveletter + ":\windows\system32\drivers\vmbus.sys")).VersionInfo.ProductVersion -eq $latestversion2008up)
        {
            $serverprops.add('IntegrationTools',"Up2Date")
        }
        else
        {
            $serverprops.add('IntegrationTools',"Outdated")
        }
    }
    else
    {
        if ((get-item ($driveletter + ":\windows\system32\drivers\vmbus.sys")).VersionInfo.ProductVersion -eq $latestversion2008up)
        {
            $serverprops.add('IntegrationTools',"Up2Date")
        }

        else
        {
            Add-WindowsPackage -PackagePath $integrationServicesCabPath -Path ($driveLetter + ":\") -ea silentlycontinue -ev 'noupdate'
            if ($noupdate)
            {
                $serverprops.add('Update',"Failed")
            }
            else
            {
                $serverprops.add('Update',"Success")
            }
        }
    }
    Dismount-VHD -Path $vhd
    New-Object -TypeName psobject -Property $serverprops
}
```