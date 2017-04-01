---
title:  "Live Migration Blocked by NPIV ports still in use Solved!"
date:   2014-05-29 12:00:00
categories: ["Hyper-V","Windows Server","Virtual Fibre Channel"]
tags: ["Hyper-V","PowerShell","Windows Server","Virtaul Fibre Channel"]
---
Working with virtual Fibre Channel on Hyper-V is really easy until you need to troubleshoot things. There doesn’t seem to be any out of the box tools to investigate and troubleshoot NPIV and vSAN related issues besides tracing. The problem with tracing at this point is that only Microsoft has the symbol files to convert the trace files into readable format which means you have to go through Microsoft support.

In my experience, virtual Fibre works pretty flawless. Every now and then though, mostly when my hosts haven’t been rebooted in some time, I am confronted with issues where the VM is not live migratable. Mostly I’m confronted with an error like: Failed to start reserving resources with Error ‘Insufficient system resources exist to complete the requested service.’ (0x800705AA).

When looking at my physical fabric (Fibre switches), I can actually see that both WWPN A and B are active at the same time. I can thus conclude that somewhere a Live Migration occurred which did not cleanup the source virtual fibre channel port on the source host. When following the physical fiber cable where the WWPN is reported on, I can actually verify that this is so. The workaround found on the internet is to simply reboot the host which will clean-up the “inactive” ports. Now this is doable until you get into a deadlock situation where VMa is running on Hosta with its inactive WWPN is still present on Hostb and vice versa. Now what? Stop the VM and reboot the host? Not in my book :)

I started investigating if there were any WMI sources to query for virtual ports and found the MSFC_FibrePortNPIVAttributes class under the “root\wmi” namespace [http://msdn.microsoft.com/en-us/library/windows/hardware/hh127630(v=vs.85).aspx](http://msdn.microsoft.com/en-us/library/windows/hardware/hh127630(v=vs.85).aspx){:target="_blank"}. So I ran ```Get-CimInstance -Namespace root\wmi -ClassName MSFC_FibrePortNPIVAttributes``` and got the following output:

```
Active              : True
InstanceName        : PCI\VEN_1077&DEV_2532&SUBSYS_3263103C&REV_02\4&2af89a5&0&0008_0
NumberVirtualPorts  : 3
VirtualPorts        : {MSFC_VirtualFibrePortAttributes, MSFC_VirtualFibrePortAttributes, MSFC_VirtualFibrePortAttributes}
WWNN                : {80, 1, 67, 128…}
WWPN                : {80, 1, 67, 128…}
```

As you can see, the object property VirtualPorts contains the virtual ports I am looking for.
When expanding the property by running ```Get-CimInstance -Namespace root\wmi -ClassName MSFC_FibrePortNPIVAttributes | select virtualports
-ExpandProperty virtualports``` I got some more information:

```
virtualports    : {MSFC_VirtualFibrePortAttributes, MSFC_VirtualFibrePortAttributes, MSFC_VirtualFibrePortAttributes}
FabricWWN       : {16, 0, 0, 192…}
FCId            : 199427
Status          : 0
Tag             : {0, 0, 0, 0…}
VirtualName     : {72, 121, 112, 101…}
WWNN            : {192, 3, 255, 0…}
WWPN            : {192, 3, 255, 110…} 
```

Now the WWPN value is a byte array from which each byte needs to be converted to HEX and appended in a STRING if you want to visually verify the WWPN with the value you will find in the VM configuration. For ease of reusability I added the converted value to the object.

```powershell
Get-CimInstance -Namespace root\wmi -ClassName MSFC_FibrePortNPIVAttributes | select virtualports -ExpandProperty virtualports | % {
    $string = “”
    $_.wwpn | %{$string = $string + $_.tostring(“X2”)}
    $_ | Add-Member -MemberType NoteProperty -Name WWPN-HEX -Value $string
    $_
}
```

Now I got the previous object enriched with the WWPN (WWPN-HEX) in a format I can compare it in.

```
virtualports    : {MSFC_VirtualFibrePortAttributes, MSFC_VirtualFibrePortAttributes, MSFC_VirtualFibrePortAttributes}
WWPN-HEX        : C003FF6EDB4A0004
FabricWWN       : {16, 0, 0, 192…}
FCId            : 199427
Status          : 0
Tag             : {0, 0, 0, 0…}
VirtualName     : {72, 121, 112, 101…}
WWNN            : {192, 3, 255, 0…}
WWPN            : {192, 3, 255, 110…} 
```

So now I have insight in all virtual ports which are active on the host. I still need a way to find out which ports are active and which ones are not. I expanded the script querying ```(Get-CimInstance -Namespace root\virtualization\v2 -ClassName Msvm_FcEndpoint).wwpn``` which enumerates all active fibre channel endpoints.

```
50014380242A5B02
50014380242A5B00
C003FFAB37C40002
C003FFAB37C40000
C003FF6EDB4A0006
C003FF6EDB4A0004
50014380242A5B02
```

Then I found the physical HBAs were also present in the result list. I made sure I excluded them by querying another class ```(Get-CimInstance -Namespace root\virtualization\v2 -ClassName Msvm_ExternalFcPort).wwpn``` and subtracting the results from the previous output.

```
C003FFAB37C40002
C003FFAB37C40000
C003FF6EDB4A0006
C003FF6EDB4A0004 
```

Now the only thing left to do is comparing the virtual ports as they are present on the vSAN against the WWPNs which are attached to running VMs. I wrote a little function to combine all these steps and allowed it to run against a range of hosts.

```powershell
function Get-FCNPIVInactivePort {
    param(
        [string[]]$computername = “.”
    )
    foreach ($c in $computername)
    {
        # physical HBA
        $pHBA = (Get-CimInstance -Namespace root\virtualization\v2 -ClassName Msvm_ExternalFcPort -ComputerName $c).wwpn
        # filter out physical hba
        [array]$vHBA = (Get-CimInstance -Namespace root\virtualization\v2 -ClassName Msvm_FcEndpoint -ComputerName $c).wwpn | %{
            if ($phba -notcontains $_)
            {
                $_
            }
        }

        Get-CimInstance -Namespace root\wmi -ClassName MSFC_FibrePortNPIVAttributes -ComputerName $c | select virtualports -ExpandProperty virtualports | % {
            $string = “”
            $_.wwpn | %{$string = $string + $_.tostring(“X2”)}
            if ($vHBA -notcontains $string)
            {
                $_ | Add-Member -MemberType NoteProperty -Name WWPN-HEX -Value $string
                $_
            }
        }
    }
}
```

The results outputted by this little function are the actual virtual ports that are still active but not actually attached to a VM. What still needs to be done is a cleanup of these ports. I wrote another little function to do that of course!

```powershell
function Remove-FCNPIVInactivePort {
    param (
        [Parameter(Mandatory=$true,
            ValueFromPipeline=$true)]
        [ValidateNotNull()]
        [ValidateNotNullOrEmpty()]
        $StalePort
    )
    process {
        (gwmi -Namespace root\wmi -Class MSFC_FibrePortNPIVMethods -ComputerName $StalePort.pscomputername).removevirtualport($staleport.wwpn)
    }
}
```

Combining the 2 functions will result in all “lingering” ports to be removed and for Live Migration to commence as expected again, no reboot required!!

```powershell
Get-FCNPIVInactivePort -computername “hyp01”,“hyp02” | Remove-FCNPIVStalePort
```

I’m thinking of making the functions more mature (built in some checks and whatif), I did this on the fly in a few hours seeking for a solution, not to make a fancy script.

When paying attention you will note that the cleanup method is invoked through get-wmiobject which I put in there because the invoke-cimmethod did not accept my arguments. I don’t know why yet but something goes wrong in the type casting of the byte array.

Use at your own risk! Always use the get function first without the remove function. Visually compare so you know for sure what you are doing (I trust these functions completely but you should make sure you do so as well!).

Sample query your environment.

```powershell
1..9 | %{Get-FCNPIVInactivePort -computername “hyp0$_“}
10..99 | %{Get-FCNPIVInactivePort -computername “hyp$_“} 
```

Sample query and remove lingering ports from your environment.

```powershell
1..9 | %{Get-FCNPIVInactivePort -computername “hyp0$_“ | Remove-FCNPIVInactivePort}
10..99 | %{Get-FCNPIVInactivePort -computername “hyp$_“ | Remove-FCNPIVInactivePort}
```

N.B. I did not use any cmdlets from the Hyper-V module because these are not installed by default (they should!). I also don’t compare off host, I figured that the host running the current VM workloads should reflect only what it knows about :)