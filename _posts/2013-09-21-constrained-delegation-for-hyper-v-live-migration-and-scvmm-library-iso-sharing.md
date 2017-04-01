---
title:  "Constrained delegation for Hyper-V live migration and SCVMM library ISO sharing"
date:   2013-09-21 12:00:00
categories: ["Hyper-V","Windows Server","SCVMM","Active Directory"]
tags: ["Hyper-V","Windows Server","Kerberos","PowerShell","SCVMM","Active Directory"]
---
When you have a lot of Hyper-V servers managing the constrained delegation required to enable remotely initiated Live Migration is a real pain.
I wrote a little PowerShell script which does it for me, including the CIFS delegation for sharing ISO files to VMs from the SCVMM library share.

Fill in your own variables where needed, run on a schedule and you're fine, never to worry about KCD again!
(For this script I assume there is some sort of naming convention on which the Hyper-V servers and clusters are recognizable, when this is not the case, you can of course utilize other sources like SCVMM to acquire the right computer names)

```powershell
$DNSSuffix = "." + (Get-ADDomain).dnsroot
$SCVMMLibrarySRV = "SCVMMLIB"
$computers = Get-ADComputer -Filter 'name -like "HYP*" -and name -notlike "HYPCLUS*"'
foreach ($c in $computers)
{
    $kcdentries = $computers | ?{$_.name -ne $c.name}
    [array]$ServiceString = @()
    foreach ($k in $kcdentries)
    {
        $ServiceString += "cifs/"+$k.name+$DNSSuffix
        $ServiceString += "cifs/"+$k.name
        $ServiceString += "Microsoft Virtual System Migration Service/"+$k.name+$DNSSuffix
        $ServiceString += "Microsoft Virtual System Migration Service/"+$k.name
    }
    $ServiceString += "cifs/"+$SCVMMLibrarySRV+$DNSSuffix
    $ServiceString += "cifs/"+$SCVMMLibrarySRV
    <# for LM KCD with Kerberos only is enough ISO sharing through SCVMM library requires protocol transition according to http://technet.microsoft.com/en-us/library/ee340124.aspx to enable protocol transition, the useraccountcontrol attribute will be changed to 16781344 #>
    Set-ADObject -Identity $C -replace @{"msDS-AllowedToDelegateTo" = $ServiceString; "userAccountControl"="16781344"}
}
```