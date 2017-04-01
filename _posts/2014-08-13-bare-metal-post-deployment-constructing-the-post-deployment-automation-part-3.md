---
title:  "Bare Metal Post-Deployment – Constructing the Post-Deployment Automation – Part 3"
date:   2014-08-13 12:00:00
categories: ["Hyper-V","SCVMM","Bare Metal Deployment","SMA","Windows Server","Virtual Fibre Channel"]
tags: ["Hyper-V","SCVMM","Bare Metal Deployment","SMA","Windows Server","Virtual Fibre Channel"]
---
This blog post is written and published on Hyper-V.nu: [https://hyper-v.nu/archives/bgelens/2014/08/bare-metal-post-deployment-constructing-the-post-deployment-automation-part-3-2/](https://hyper-v.nu/archives/bgelens/2014/08/bare-metal-post-deployment-constructing-the-post-deployment-automation-part-3-2/){:target="_blank"}

This blog post series consists of 5 parts:

1. [Part 1: Introduction and Scenario](http://bgelens.nl/bare-metal-post-deployment-introduction-and-scenario-part-1/)
2. [Part 2: Pre-Conditions  (this blog post)](http://bgelens.nl/bare-metal-post-deployment-pre-conditions-part-2/)
3. [Part 3: Constructing the Post-Deployment automation](http://bgelens.nl/bare-metal-post-deployment-constructing-the-post-deployment-automation-part-3/)
4. [Part 4: Constructing the Post-Deployment automation continued](http://bgelens.nl/bare-metal-post-deployment-constructing-the-post-deployment-automation-continued-part-4/)
5. [Part 5: Running the Post-Deployment automation and Bonus](http://bgelens.nl/bare-metal-post-deployment-running-the-post-deployment-automation-and-bonus-part-5/)

# Constructing the Post-Deployment automation

To make the process extensible we use a master runbook to determine the common tasks that need to be executed on all hosts and decide where to call which child runbook when tasks become specific. When a custom resource script is run, the script is described after the runbook.

This chapter will be split into 2 blog posts. Let’s start with the master runbook.

# Master Runbook “Run-HyperVPostDeployment”.

At first the runbook will acquire the credentials and FQDN needed to connect to the SCVMM environment. Then it will pass the data to the Get-HyperVHosts child runbook and will ask it to deliver Hyper-V hosts which are ready for post deployment. For every host which is returned, a connectivity check is performed against WinRM. When a Host is responsive, it will be placed in maintenance mode and the hardware type will be queried from it to be stored together with the SCVMM host object in memory. Every non-active host is filtered out. All active hosts will then run the following process in parallel (2 at a time because of the throttlelimit set at the $throttlelimit variable, 5 is the maximum limitation of workflow):

* Group policy update (make sure all GPO firewall rules are applied)
* Filtering of hardware type and environment
* Processing specific child runbook (if ran successfully continue, else break for Host)
* Perform the NIC registry fix
* Switch off maintenance mode
* Marked “Finished” with post deployment (Post Deployment Status custom property = “Finished”)

![](/images/2014-08/Bare-Metal-Post-Deployment.png)

```powershell
workflow Run-HyperVPostDeployment
{
    [PSCredential]$creds = Get-AutomationPSCredential -Name 'SMA SCVMM Service Account'
    [string]$VMMServer = Get-AutomationVariable -Name 'VMMServer'
    [string]$smawebserver = Get-AutomationVariable -Name 'SMAWebServer'
    [int]$smawebserverport = "9090"
    [int]$throttlelimit = 2
    inlinescript {
        import-module virtualmachinemanager
        get-scvmmserver -computername $using:vmmserver -credential $using:creds | out-null
    }
    $PostDeployHosts = Get-HyperVHosts -Creds $creds -vmmserver $vmmserver -ReadyForPostDeploy $true
    $activenodes = @()
        foreach ($H in $PostDeployHosts)
        {
            if (Test-NetConnection -ComputerName $H.computername -CommonTCPPort WINRM -InformationLevel Quiet)
            {
                inlinescript
                {
                    Disable-VMHost -VMHost $using:H.computername | out-null
                }
                $H | Add-Member -MemberType NoteProperty -Name Hardware -Value ((Get-CimInstance -CimSession (New-CimSession -ComputerName $h.computername -Credential $creds) -ClassName Win32_ComputerSystem).model.toupper())
                $activenodes += $H
            }
            else
            {
                write-error "$H is not reachable and will be excluded"
            }
        }
    foreach –parallel -throttlelimit $throttlelimit ($N in $activenodes)
    {
            inlinescript
            {
                $scriptSetting = New-SCScriptCommandSetting -WarnAndContinueOnMatch -AlwaysReboot $false
                Invoke-SCScriptCommand -Executable "cmd.exe" -TimeoutSeconds 1800 -CommandParameters "/q /c gpupdate /force" -VMHost $using:N.computername -JobVariable gpupdate -RunAsynchronously | Out-Null
                while ($gpupdate.status -eq "Running") {Start-Sleep 5}
            }
            switch -CaseSensitive ($N.Hardware)
            {
                "PROLIANT BL460C GEN8" {
                    $runbook = "Config-BL460CGen8"
                    if ($N.VMHostGroup -eq "All Hosts\Production")
                    {
                        $params = @{
                            'vmhost'=$N.fqdn;
                            'type'='Production'
                        }
                    }
                    elseif ($N.VMHostGroup -eq "All Hosts\Test")
                    {
                        $params = @{
                            'vmhost'=$N.fqdn;
                            'type'='Test'
                        }
                    }
                    else
                    {
                        #No postdeploy has been created yet
                        $ChildRunbookExitMessage = "Failed: BL460C Gen8 unknown host environment type"
                    }
                    }
                "PROLIANT DL380P GEN8" {
                $runbook = "Config-DL380G8"
                    if ($N.VMHostGroup -eq "All Hosts\Production")
                    {
                        $params = @{
                            'vmhost'=$N.fqdn;
                            'type'='Disaster'
                        }
                    }
                    elseif ($N.VMHostGroup -eq "All Hosts\Test")
                    {
                        $params = @{
                            'vmhost'=$N.fqdn;
                            'type'='Test'
                        }
                    }
                    else
                    {
                        #No postdeploy has been created yet
                        $ChildRunbookExitMessage = "Failed: DL380P GEN8 unknown host environment type"
                    }
                    }
                default {$ChildRunbookExitMessage = "Failed: unknown hardware type"}
            }

            if ($params)
            {
                $job = Start-SmaRunbook -WebServiceEndpoint $smawebserver -Port $smawebserverport -Name $runbook -Parameters $params
                do
                {
                    $jobinfo = Get-SmaJob -WebServiceEndpoint $smawebserver -Port $smawebserverport -Id $job
                }
                while ($jobinfo.jobstatus -ne "Completed")
                $output = Get-SmaJobOutput -WebServiceEndpoint $smawebserver -Port $smawebserverport -Id $job -Stream Output
                $ChildRunbookExitMessage = $output.streamtext
            }
            if ($ChildRunbookExitMessage -like "Failed*")
            {
                Write-Output "Failed Post-Deployment: $N - $ChildRunbookExitMessage"
            }
            else
            {
                $finalstatus = inlinescript
                {

                    $scriptSetting = New-SCScriptCommandSetting -FailOnMatch -AlwaysReboot $false
                    $LibraryResource = Get-SCCustomResource -Name "Hostdeploy.cr"
                    $RunAsAccount = Get-SCRunAsAccount -Name "SCVMM Admin Run As Account"
                    Invoke-SCScriptCommand -Executable "%SystemRoot%\system32\WindowsPowerShell\v1.0\powershell.exe" -TimeoutSeconds 300 -CommandParameters "-file nic_registryfix.ps1" -VMHost $using:N.computername -LibraryResource $LibraryResource -RunAsAccount $RunAsAccount -ScriptCommandSetting $scriptSetting -RunAsynchronously -JobVariable regfix | Out-Null
                    while ($regfix.status -eq "Running") {start-sleep 5}
                    if ($regfix.status -eq "Failed")
                    {
                        return "Failed at Registry Fix stage"
                    }
                    else
                    {
                        Restart-SCVMHost -VMHost $using:N.computername -Confirm:$false | out-null
                        Read-SCVMHost $using:N.computername | out-null                                
                        $CustomProperty = Get-SCCustomProperty -Name "Post Deployment Status"
                        Set-SCCustomPropertyValue -InputObject (get-scvmhost $using:N.computername) -CustomProperty $CustomProperty -value "Finished" | Out-Null
                        Enable-VMHost -vmhost $using:N.computername | out-null
                        return "Success"
                    }
                }
                if ($finalstatus -like "Failed*")
                {
                
                    write-output "Failed Post-Deployment: $N - $finalstatus"
                }  
                write-output "Successfully Finished Post-Deployment: $N"
               }
    } # foreach parallel    
}
```

# Custom Resource: Nic_registryfix.ps1

Nic_registryfix.ps1 will run on all hosts and it will handle the following tasks:

* Query for all VM Switches
* Lookup the NIC on which a switch is bound
* If the NIC is an team multiplexor NIC, lookup all team member adapters
* For all NICs found, add registry items to disable DNS dynamic update and DHCP.

```powershell
$GUIDS = @()
Get-VMSwitch | %{
$NETLBFONIC = Get-NetAdapter -InterfaceDescription $_.NetAdapterInterfaceDescription
$GUIDS += $NETLBFONIC.interfaceguid
    if ($NETLBFONIC.InterfaceDescription -like "*multiplexor*")
    {
        Get-NetLbfoTeam $NETLBFONIC.name | get-netlbfoteammember | %{
        $GUIDS += (get-netadapter -name $_.name).interfaceguid
        }
    }
}

foreach ($G in $GUIDS)
{
    if (Get-ItemProperty -Path HKLM:\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters\Interfaces\$G -name DisableDynamicUpdate -ErrorAction SilentlyContinue)
    {
        set-ItemProperty -Path HKLM:\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters\Interfaces\$G -Name DisableDynamicUpdate -Value 1
    }
    else
    {
        New-ItemProperty -Path HKLM:\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters\Interfaces\$G -Name DisableDynamicUpdate -Value 1
    }
    Set-ItemProperty -Path HKLM:\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters\Interfaces\$G -Name EnableDHCP -Value 0
}
```

# Child Runbook “Get-HyperVHosts”.

Get-HyperVHosts runbook queries SCVMM for all Hyper-V hosts and either checks if the “Post Deployment Status” custom attribute is empty (-ReadyForPostDeploy $true) or has the value provided by the invoker (e.g. -PostDeploymentStatus “Finished”). If the non-mandatory parameters are omitted, it will return all Hyper-V host objects. This runbook can be called directly or as a child runbook. In this case it will be called directly with the inline method (nested).

```powershell
$PostDeployHosts = Get-HyperVHosts -Creds $creds -vmmserver $vmmserver -ReadyForPostDeploy $true
```

I’ll be using multiple methods to show the possibilities in SMA. For more info see: [http://blogs.technet.com/b/orchestrator/archive/2014/01/10/sma-capabilities-in-depth-runbook-input-output-and-nested-runbooks.aspx](http://blogs.technet.com/b/orchestrator/archive/2014/01/10/sma-capabilities-in-depth-runbook-input-output-and-nested-runbooks.aspx){:target="_blank"}

```powershell
workflow Get-HyperVHosts
{
    param (
        [Parameter(Mandatory=$true)]
        [PSCredential]$Creds,
        [Parameter(Mandatory=$true)]
        [string]$VMMServer,
        [bool]$ReadyForPostDeploy,
        [string]$PostDeploymentStatus 
    )
    InlineScript {
        [string]$CustomPropertyName = "Post Deployment Status"
        import-module virtualmachinemanager
        get-scvmmserver -computername $using:vmmserver -credential $using:creds | out-null
        $hosts = get-scvmhost
        if ($using:PostDeploymentStatus -or $using:ReadyForPostDeploy)
        {
            $CustomProperty = Get-SCCustomProperty -Name $CustomPropertyName
            foreach ($obj in $hosts)
            {
                $Property = Get-SCCustomPropertyValue -InputObject $Obj -CustomProperty $CustomProperty
                if ($using:postdeploymentstatus -eq "New"-or $using:ReadyForPostDeploy -and $Property -eq $null)
                {
                    $obj
                }
                
                elseif ($Property.value -eq $using:postdeploymentstatus)
                {
                    $obj
                }
            }
        }
        else
        { 
            $hosts
        }                      
    }    
}
```