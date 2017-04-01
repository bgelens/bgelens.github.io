---
title:  "Hyper-V vCPU to Core ratio script"
date:   2014-01-23 12:00:00
categories: ["Hyper-V"]
tags: ["Hyper-V","PowerShell"]
---
I wrote a little script to figure out the current vCPU to physical core ratio on Hyper_V machines.

Please pay attention, this script can only run on 2012R2 / 8.1 due to the ```Test-NetConnection``` cmdlet which is used to verify winRM connectivity. This can of course be easily replaced with a simple test-connection to make it friendly for older windows versions.

There is a lot of information out there which is just simply wrong! Or I am wrong, which isn’t unlikely either because I don’t work for Microsoft. So this is my interpretation from Ben Armstrong’s explanation that hyper threaded cores don’t count for Hyper-V, so they shouldn’t be used in your calculations either.

I used CIM to call for the required information, so no Hyper-V Powershell module is necessary. I made it a workflow so if you need to check like 500 hosts, they will be checked in parallel (I believe it will do 5 simultaneous).

HTH!
```powershell
workflow Get-CoreRatio {
    Param(
        [Parameter(Mandatory=$false)]
        [string[]]$ComputerName
        )
    if (!($ComputerName))
    {
        $ComputerName = $env:COMPUTERNAME
    }
    foreach –parallel ($c in $ComputerName)
    {
        sequence 
        {
            inlinescript
            {
                $C2 = $using:C
                $c2 = $c2.toupper()
                $HyperVInfo = New-Object System.Collections.Specialized.OrderedDictionary
                $HyperVInfo.add("Hyper-V Host",$C2)
                $HyperVInfo.add('Number of physical cores',"")
                $HyperVInfo.add('Number of VMs',"")
                $HyperVInfo.add('Number of vCPUs',"")
                $HyperVInfo.add('vCPU to Core ratio',"")

                if (Test-NetConnection $C2 -CommonTCPPort WINRM -InformationLevel Quiet -WarningAction SilentlyContinue)
                {
                    $HyperVInfo."Number of physical cores" = (Get-CimInstance -CimSession $C2 Win32_Processor | Measure-Object -Property NumberOfCores -Sum).sum
                    $HyperVInfo."Number of VMs" = (Get-CimInstance -CimSession $C2 -Namespace root\virtualization\v2 -ClassName Msvm_ComputerSystem -Filter 'Caption="Virtual Machine"').count
                    $HyperVInfo."Number of vCPUs" = (Get-CimInstance -CimSession $C2 -Namespace root\virtualization\v2 -ClassName CIM_Processor -Filter 'Role="Virtual Processor"').count
                    $HyperVInfo."vCPU to Core ratio" = [string]($HyperVInfo."Number of vCPUs" / $HyperVInfo."Number of physical cores") + ":1"
                }
                else
                {
                    $HyperVInfo."Number of physical cores" = "Unreachable"
                    $HyperVInfo."Number of VMs" = "Unreachable"
                    $HyperVInfo."Number of vCPUs" = "Unreachable"
                    $HyperVInfo."vCPU to Core ratio" = "Unreachable"
                }
                New-Object -TypeName psobject -Property $HyperVInfo
            }
        }
    }
}
Get-CoreRatio 
Get-CoreRatio -ComputerName HV1
Get-CoreRatio -ComputerName HV1,HV2,HVx 
```