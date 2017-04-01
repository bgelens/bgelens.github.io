---
title:  "Discover Numa Misalligned VMs and report on amount and percentage of remote memory"
date:   2014-01-31 12:00:00
categories: ["Hyper-V"]
tags: ["Hyper-V","PowerShell","NUMA"]
---
I wrote a little script today to check for Numa misalignment of my VMs. 

We have a lot of SQL VMs which when misalignment occurs could degrade in performance because if this.
The script will only show VMs which use remote memory and are thus misaligned. As information you get the total amount of pages used by the VM, the amount of pages which are in remote memory and the percentage this remote memory is making up of the total amount of pages.

HTH!

**Update 1:** added remote and total memory in MB to object properties (based on 4096bytes per memory page).

```powershell
function Get-NumaMisalignedVM {
    Param(
        [Parameter(Mandatory=$true)]
        [string[]]$ComputerName
        )
    foreach ($comp in $ComputerName)
    {
        $counter = Get-Counter "\Hyper-V VM VID Partition(*)\*" -ComputerName $Comp | select CounterSamples -ExpandProperty CounterSamples
        foreach ($c in $counter)
        {
            if (($c | ?{$_.path -like "*remote*"}).cookedvalue -gt 0 -and $c.instancename -ne "_total")
            {
                $misalligned = $counter |?{$_.instancename -eq $c.instancename}
                $misallignedobj = New-Object System.Collections.Specialized.OrderedDictionary
                $misallignedobj.add("VM",($c.instancename).toupper())
                $misallignedobj.add("Host",$comp.ToUpper())
                $misallignedobj.add('Remote Physical Pages',[int]($misalligned | ?{$_.path -like "*remote physical pages*"}).cookedvalue)
                $misallignedobj.add('Physical Pages Allocated',[int]($misalligned | ?{$_.path -like "*physical pages allocated*"}).cookedvalue)
                $misallignedobj.add('Remote memory in MB',[int](($misallignedobj."Remote Physical Pages" * 4096) / 1048576))
                $misallignedobj.add('Physical in MB',[int](($misallignedobj."Physical Pages Allocated" * 4096) / 1048576))
                $misallignedobj.add('Remote Memory %',("{0:N2}" -f (($misallignedobj."Remote Physical Pages" / $misallignedobj."Physical Pages Allocated") * 100)))
                New-Object -TypeName psobject -Property $misallignedobj
            }
        }
    }
}
Get-NumaMisalignedVM -ComputerName hyper-v

VM                       : VM1
Host                     : HYPER-V
Remote Physical Pages    : 2709963
Physical Pages Allocated : 6293512
Remote memory in MB      : 10586
Physical in MB           : 24584
Remote Memory %          : 43,06

VM                       : VM2
Host                     : HYPER-V
Remote Physical Pages    : 1727675
Physical Pages Allocated : 8390664
Remote memory in MB      : 6749
Physical in MB           : 32776
Remote Memory %          : 20,59
```

NB. As with most of my postings here, the script is not a beauty but it is functional. I wrote this tiny function in 10 minutes. I intend to write a module of functions in the near future with more developed code when I have the time.