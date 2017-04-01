---
title:  "SMA Runbook: Enable Kerberos Constrained delegation for Hyper-V Live Migration and ISO sharing in AD and SCVMM"
date:   2014-08-29 12:00:00
categories: ["Hyper-V","Windows Server","SMA"]
tags: ["Hyper-V","Windows Server","Kerberos","SMA","SCVMM"]
---
Today I'll share a little SMA runbook which enables Kerberos Constrained Delegation for all your Hyper-V hosts which are available in SCVMM.
It does this by configuring the Active Directory side of things and the SCVMM side of things and it is based on my earlier post which can be found [here](http://bgelens.nl/constrained-delegation-for-hyper-v-live-migration-and-scvmm-library-iso-sharing/).

When a tasks fails, an error will be logged in the SMA runbook history.

For this runbook to work you need to register some SMA assets:
String Variable VMMServer and DomainController
PSCredentials "AD Service Account" and "SCVMM Service Account"

Note that the credentials added as asset should have enough privileges to perform the tasks!

The runbook enables live migration for all subnets which can be adjusted quite easily if needed.
I've named the runbook **Reset-HyperVConstrainedDelegation** because this is exactly what it does, it RESETS the constrained delegation (overwrites) to include all known Hyper-V hosts and library servers.

Hope this helps you! Ben

```powershell
workflow Reset-HyperVConstrainedDelegation
{
    $vmmcreds = Get-AutomationPSCredential -Name 'SCVMM Service Account'
    $vmmserver = Get-AutomationVariable -Name 'VMMServer'
    $domaincontroller = Get-AutomationVariable -Name 'DomainController'
    $adcreds = Get-AutomationPSCredential -Name 'AD Service Account'
    
    $hosts = inlinescript {
        Import-Module virtualmachinemanager
        get-scvmmserver -computername $using:vmmserver | out-null
        get-scvmhost
    } -pscomputername $vmmserver -pscredential $vmmcreds
    
    $libraryservers = inlinescript {
        Get-SCLibraryServer
    } -pscomputername $vmmserver -pscredential $vmmcreds

    foreach -parallel ($h in $hosts) {
        inlinescript {
            Import-Module activedirectory
            $kcdentries = $using:hosts | ?{$_.fqdn -ne $using:h.fqdn}
            [array]$ServiceString = @()
            foreach ($k in $kcdentries)
            {
                $ServiceString += "cifs/"+$k.fqdn
                $ServiceString += "cifs/"+$k.computername
                $ServiceString += "Microsoft Virtual System Migration Service/"+$k.fqdn
                $ServiceString += "Microsoft Virtual System Migration Service/"+$k.computername
            }
            foreach ($l in $using:libraryservers)
            {
                $ServiceString += "cifs/"+$l.fqdn
                $ServiceString += "cifs/"+$l.computername
            }
            [string]$filter = 'samaccountname -like "' + $h.computername + "$" + '"'
            try
            {
                Get-ADObject -Filter $filter | Set-ADObject -replace @{"msDS-AllowedToDelegateTo" = $ServiceString; "userAccountControl"="16781344"} -ErrorAction Stop
            }
            catch
            {
                $fqdn = $using:h.fqdn
                Write-Error "Failed AD Configuration for host $fqdn"
            }
        } -pscomputername $domaincontroller -pscredential $adcreds
    }

    foreach -parallel ($h in $hosts) {
        inlinescript {
            try
            {
                Set-SCVMHost -VMHost $using:h.fqdn -EnableLiveMigration $true -MigrationPerformanceOption "UseCompression" -MigrationAuthProtocol "Kerberos" -UseAnyMigrationSubnet $true -ErrorAction Stop| out-null
            }
            catch
            {
                $fqdn = $using:h.fqdn
                Write-Error "Failed SCVMM Configuration for host $fqdn"
            }
        } -pscomputername $vmmserver -pscredential $vmmcreds
    }
}
```