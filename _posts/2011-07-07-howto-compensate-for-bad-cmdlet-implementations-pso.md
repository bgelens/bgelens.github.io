---
title:  "Howto compensate for bad CMDlet implementations! PSO"
date:   2011-07-07 12:00:00
categories: ["Active Directory"]
tags: ["Windows Server","Active Directory","PowerShell"]
---
One of the finest new features since server 2008 is the ability to create multiple password policies.
Server 2008R2 takes this one step further by allowing its implementation through PowerShell CMDlets.
One biggie though... If you try to set the lockout duration to indefinitely (administrative intervention necessary), you just can't do it.

Here is how to create and compensate :-)

```powershell
Import-Module ActiveDirectory -ea silentlycontinue
$DN = (Get-ADDomain).DistinguishedName
New-ADFineGrainedPasswordPolicy -Name "PasswordPSO" -Precedence 10 -ComplexityEnabled $true -Description "Lockout forever!" -DisplayName "Lockout forever PSO" -LockoutDuration "0.12:00:00" -LockoutObservationWindow "0.01:00:00" -LockoutThreshold 3 -MaxPasswordAge "45.00:00:00" -MinPasswordAge "30.00:00:00" -MinPasswordLength 12 -PasswordHistoryCount 12 -ReversibleEncryptionEnabled $false
Set-ADObject "CN=PasswordPSO,CN=Password Settings Container,CN=System,$DN" -replace @{'msDS-LockoutDuration'='-9223372036854775808'}
Add-ADFineGrainedPasswordPolicySubject PasswordPSO -Subjects Groupname
```

The trick here is ```-9223372036854775808``` is the equivalent of the large integer NEVER when set through ADSIEDIT. Nice huh those GUI tools that hide everything :-)