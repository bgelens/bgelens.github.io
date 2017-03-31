---
title:  "RODC Allowed Password Replication made easy!"
date:   2011-07-06 12:00:00
categories: ["Ben Gelens","Windows Server"]
tags: ["Ben Gelens","RODC","Windows Server","Active Directory","PowerShell"]
---
What is the use of a RODC in an environment that is totally centralized in a data center?
Answer: A highly controllable and secure authentication (proxy) for remote users!

Off course you want to do this redundant and so you create a site with multiple RODC's.
One of the functional requirements will probably be that, when connectivity with the RWDC hub site is gone, at least the RODC should be manageable and a RODC Administrator should be able to log in locally (even when he never ever logged in before!).
While figuring this out, sooner or later you will find that it isn't easy to manage the locally cached secrets (passwords) because every RODC will act on its own and thus has its own cached secrets.

What you need to do to make this simple is follow the best practise of using the Allowed RODC Password Replication Group group to manage which secrets will be available on the RODC's and... Use this very simple script (modify it to your needs off course)!

```powershell
Import-Module ActiveDirectory -ea silentlycontinue
$DN = (Get-ADDomain).DistinguishedName
$pdc = Get-ADDomainController -discover -Service "PrimaryDC"
$rodcs = Get-ADDomainController -Filter { isReadOnly -eq $true}
Get-ADGroupMember -Recursive "CN=Allowed RODC Password Replication Group,CN=Users,$DN" | %{
    $obj = $_.distinguishedName
    $rodcs | %{
        Repadmin /rodcpwdrepl $_.name $pdc.name $obj
    }
}
```

This script will replicate secrets from the PDC (you can modify this off course!) to every RODC.
It will query the Allowed RODC Password Replication Group group recursivly to determine which secrets to replicate!

Easy! Hope you enjoy!