---
title:  "Bare Metal Post-Deployment – Constructing the Post Deployment Automation (continued) Part 4"
date:   2014-08-14 12:00:00
categories: ["Hyper-V","SCVMM","Bare Metal Deployment","SMA","Windows Server","Virtual Fibre Channel"]
tags: ["Hyper-V","SCVMM","Bare Metal Deployment","SMA","Windows Server","Virtual Fibre Channel"]
---
This blog post is written and published on Hyper-V.nu: [https://hyper-v.nu/archives/bgelens/2014/08/bare-metal-post-deployment-constructing-the-post-deployment-automation-continued-part-4/](https://hyper-v.nu/archives/bgelens/2014/08/bare-metal-post-deployment-constructing-the-post-deployment-automation-continued-part-4/){:target="_blank"}

This blog post series consists of 5 parts:

1. [Part 1: Introduction and Scenario](http://bgelens.nl/bare-metal-post-deployment-introduction-and-scenario-part-1/)
2. [Part 2: Pre-Conditions  (this blog post)](http://bgelens.nl/bare-metal-post-deployment-pre-conditions-part-2/)
3. [Part 3: Constructing the Post-Deployment automation](http://bgelens.nl/bare-metal-post-deployment-constructing-the-post-deployment-automation-part-3/)
4. [Part 4: Constructing the Post-Deployment automation continued](http://bgelens.nl/bare-metal-post-deployment-constructing-the-post-deployment-automation-continued-part-4/)
5. [Part 5: Running the Post-Deployment automation and Bonus](http://bgelens.nl/bare-metal-post-deployment-running-the-post-deployment-automation-and-bonus-part-5/)

# Constructing the Post-Deployment automation (continued)

This blog post will describe the model and configuration specific part of the automation which is called upon the master runbook described at the previous post.

# Child Runbook “Config-BL460CGen8”.

Config-BL460CGen8 runbook is called as a child runbook when the Hyper-V host involved is a BL460C Gen8 blade. The runbook is not started inline but is started with its own job by using the Start-SmaRunbook command (my preferred method). Differentiation between configurations from different environments are made through the Type parameter.

The runbook will put the host through the following process:

* Implement Fully Converged networking including MAC address fix (bug in SCVMM is circumvented).
    * Create the logical Infra switch on a designated Infra team NIC which is not used for management.
    * Run bl460c_converged.ps1 on the host (for more details see the script below).
    * Refreshes SCVMM host information.
* Creates the VM logical switch on designated VM team NICs.
* Add LM and CSV vNICs to Infra switch.
* Configures live migration settings.
* Run Hostdeploy scripts from the Hostdeploy custom resource (for more details see the script section).
    * Postdeploy.ps1
    * BL460_npiv.ps1
        * Reboots the host for NPIV to become available
    * BL460_vmsan.ps1
    * BL460_xxxx_vmq.ps1

The script will return a Success statement to the master runbook when the entire child runbook has run successfully. If something goes wrong during the child runbook process, the process is terminated for the hosts and a Failed at stage … statement is returned to the master runbook (effectively terminating the process entirely for the host).

![](/images/2014-08/Bare-Metal-Post-Deployment-2.png)

```powershell
workflow Config-BL460CGen8
{
    [OutputType([string])]
    param(
        [string]$VMHost,
        [string]$type
    )
    $creds = Get-AutomationPSCredential -Name 'SMA SCVMM Service Account'
    $VMMServer = Get-AutomationVariable -Name 'VMMServer'
    $statuscode = inlinescript
    {
        import-module virtualmachinemanager
        get-scvmmserver -computername $using:vmmserver -credential $using:creds | out-null
        $scriptSetting = New-SCScriptCommandSetting -FailOnMatch -AlwaysReboot $false
        $LibraryResource = Get-SCCustomResource -Name "Hostdeploy.cr"
        $RunAsAccount = Get-SCRunAsAccount -Name "SCVMM Admin Run As Account"
        $vmHost = Get-SCVMHost -ComputerName $using:vmhost
        Read-SCVMHost $vmHost | out-null
        if ($using:Type -eq "Test")
        {
            $configarray = @{
            VMSWitchLOM="*554M*"
            InfraUplinkPortProfileSet="BL460C Gen8 Infra (Dynamic)"
            VMUplinkPortProfileSet="BL460C Gen8 VM Test (Dynamic)"
            VMQIndirectionTableScript="-file bl460_test_vmq.ps1"
            }
        }
        elseif ($using:Type -eq "Production")
        {
            $configarray = @{
            VMSWitchLOM="*554M*"
            InfraUplinkPortProfileSet="BL460C Gen8 Infra (Dynamic)"
            VMUplinkPortProfileSet="BL460C Gen8 VM Production (Dynamic)"
            VMQIndirectionTableScript="-file bl460_prod_vmq.ps1"
            }
        }
        else
        {
            return "Failed: No valid type defined (Test/Production?)"
        }

        $jobgroup = [guid]::NewGuid()
        $switchnic = Get-SCVMHostNetworkAdapter -VMHost $vmHost |? {$_.name -like $configarray.VMSWitchLOM -and $_.BDFLocationInformation -like "*function 1"}
        $uplinkPortProfileSet = Get-SCUplinkPortProfileSet -Name $configarray.InfraUplinkPortProfileSet
        Set-SCVMHostNetworkAdapter -VMHostNetworkAdapter $switchnic -UplinkPortProfileSet $uplinkPortProfileSet -JobGroup $jobgroup
        $logicalSwitch = Get-SCLogicalSwitch -Name "Infra"
        New-SCVirtualNetwork -VMHost $vmHost -VMHostNetworkAdapters $switchnic -LogicalSwitch $logicalSwitch -JobGroup $jobgroup
        Set-SCVMHost -VMHost $vmHost -JobGroup $jobgroup -RunAsynchronously -JobVariable infrateam | out-null
        while ($infrateam.status -eq "Running") {start-sleep 5}
        if ($infrateam.status -eq "Failed")
        {
            return "Failed at initial Infra Team stage"
        }
        Invoke-SCScriptCommand -Executable "%SystemRoot%\system32\WindowsPowerShell\v1.0\powershell.exe" -TimeoutSeconds 300 -CommandParameters "-file bl460_converged.ps1" -VMHost $vmhost -LibraryResource $LibraryResource -RunAsAccount $RunAsAccount -ScriptCommandSetting $scriptSetting -JobVariable converged -RunAsynchronously | out-null
        while ($converged.status -eq "Running") {start-sleep 5}
        if ($converged.status -eq "Failed")
        {
            return "Failed at converged stage"
        }
        Read-SCVMHost $vmHost | out-null
        $ErrorActionPreference = "stop"
        try
        {
            $jobgroup = [guid]::NewGuid()
            $networkAdapters = Get-SCVMHostNetworkAdapter -VMHost $vmHost |? {$_.BDFLocationInformation -like "*function 1" -or $_.BDFLocationInformation -like "*function 0" -and $_.name -like $configarray.VMSWitchLOM }
            Set-SCVMHostNetworkAdapter -VMHostNetworkAdapter $networkAdapters[0] -AvailableForPlacement $false -UsedForManagement $true -JobGroup $jobgroup
            Set-SCVMHostNetworkAdapter -VMHostNetworkAdapter $networkAdapters[1] -AvailableForPlacement $false -UsedForManagement $true -JobGroup $jobgroup
            $vmswitchnics = Get-SCVMHostNetworkAdapter -VMHost $vmHost| ? {$_.BDFLocationInformation -like "*function 2" -or $_.BDFLocationInformation -like "*function 3" -and $_.name -like $configarray.VMSWitchLOM }
            $VMUplinkPortProfileSet = Get-SCUplinkPortProfileSet -Name $configarray.VMUplinkPortProfileSet
            $VMlogicalSwitch = Get-SCLogicalSwitch -Name "VM Switch"
            Set-SCVMHostNetworkAdapter -VMHostNetworkAdapter $vmswitchnics[0] -UplinkPortProfileSet $VMUplinkPortProfileSet -JobGroup $jobgroup
            Set-SCVMHostNetworkAdapter -VMHostNetworkAdapter $vmswitchnics[1] -UplinkPortProfileSet $VMUplinkPortProfileSet -JobGroup $jobgroup
            New-SCVirtualNetwork -VMHost $vmHost -VMHostNetworkAdapters $vmswitchnics -LogicalSwitch $VMlogicalSwitch -JobGroup $jobgroup
            Set-SCVMHost -VMHost $vmHost -JobGroup $jobgroup -RunAsynchronously -JobVariable vmswitch | out-null
        }
        catch
        {
            return "Failed at vmswitch build job stage"
        }
        while ($vmswitch.status -eq "Running") {start-sleep 5}
        if ($vmswitch.status -eq "Failed")
        {
            return "Failed at vmswitch implementation stage"
        }
        try
        {
            $jobgroup = [guid]::NewGuid()
            $infralogicalSwitch = Get-SCLogicalSwitch -Name "infra"
            $vNICPortClassification = Get-SCPortClassification -Name "Host management"
            $vNic = Get-SCVirtualNetworkAdapter -VMHost $vmHost |?{$_.name -eq "mgmt"}
            Set-SCVirtualNetworkAdapter -VirtualNetworkAdapter $vNic -JobGroup $jobgroup -PortClassification $vNICPortClassification
            $CSVNetwork = Get-SCVMNetwork -Name "CSV + Heartbeat"
            $vmSubnet = Get-SCVMSubnet -VMNetwork $CSVNetwork -Name "CSV"
            $vNICPortClassification = Get-SCPortClassification -Name "Host Cluster Workload"
            $ipV4Pool = Get-SCStaticIPAddressPool -Name "CSV"
            New-SCVirtualNetworkAdapter -VMHost $vmHost -Name "CSV" -VMNetwork $csvNetwork -LogicalSwitch $infralogicalSwitch -JobGroup $jobgroup -VMSubnet $vmSubnet -PortClassification $vNICPortClassification -IPv4AddressType "Static" -IPv4AddressPool $ipV4Pool -MACAddressType "Static" -MACAddress "00:00:00:00:00:00"
            $LMNetwork = Get-SCVMNetwork -Name "Live Migration"
            $vmSubnet = Get-SCVMSubnet -VMNetwork $LMNetwork -Name "Live Migration"
            $vNICPortClassification = Get-SCPortClassification -Name "Live migration workload"
            $ipV4Pool = Get-SCStaticIPAddressPool -Name "Live Migration"
            New-SCVirtualNetworkAdapter -VMHost $vmHost -Name "LM" -VMNetwork $LMNetwork -LogicalSwitch $infralogicalSwitch -JobGroup $jobgroup -VMSubnet $vmSubnet -PortClassification $vNICPortClassification -IPv4AddressType "Static" -IPv4AddressPool $ipV4Pool -MACAddressType "Static" -MACAddress "00:00:00:00:00:00"
            Set-SCVMHost -VMHost $vmHost -JobGroup $jobgroup -RunAsynchronously -JobVariable vNIC | out-null
        }
        catch
        {
            return "Failed at vNIC build job stage"
        }
        while ($vNIC.status -eq "Running") {start-sleep 5}
        if ($vNIC.status -eq "Failed")
        {
            return "Failed at vNIC implementation stage"
        }
        $ErrorActionPreference = "continue"
        $LiveMigrationSubnets = @("10.10.10.0/24")
        Set-SCVMHost -VMHost $vmHost -RunAsynchronously -LiveStorageMigrationMaximum "4" -EnableLiveMigration $true -LiveMigrationMaximum "4" -MigrationPerformanceOption "UseCompression" -MigrationAuthProtocol "Kerberos" -UseAnyMigrationSubnet $false -MigrationSubnet $LiveMigrationSubnets -JobVariable migration | out-null
        while ($migration.status -eq "Running") {start-sleep 5}
        if ($migration.status -eq "Failed")
        {
            return "Failed at migration settings stage"
        }
        Invoke-SCScriptCommand -Executable "%SystemRoot%\system32\WindowsPowerShell\v1.0\powershell.exe" -TimeoutSeconds 2100 -CommandParameters "-file postdeploy.ps1" -VMHost $vmHost -LibraryResource $LibraryResource -RunAsAccount $RunAsAccount -ScriptCommandSetting $scriptSetting -RunAsynchronously -JobVariable postdeploy| out-null
        while ($postdeploy.status -eq "Running") {start-sleep 5}
        if ($postdeploy.status -eq "Failed")
        {
            return "Failed at generic postdeployment stage"
        }
        Invoke-SCScriptCommand -Executable "%SystemRoot%\system32\WindowsPowerShell\v1.0\powershell.exe" -TimeoutSeconds 120 -CommandParameters "-file bl460_npiv.ps1" -VMHost $vmHost -LibraryResource $LibraryResource -RunAsAccount $RunAsAccount -ScriptCommandSetting $scriptSetting -RunAsynchronously -JobVariable npiv| Out-Null
        while ($npiv.status -eq "Running") {start-sleep 5}
        if ($npiv.status -eq "Failed")
        {
            return "Failed at enabling NPIV stage"
        }
        start-sleep 30
        Restart-SCVMHost -VMHost $vmhost -Confirm:$false | out-null
        Invoke-SCScriptCommand -Executable "%SystemRoot%\system32\WindowsPowerShell\v1.0\powershell.exe" -TimeoutSeconds 1800 -CommandParameters "-file bl460_vmsan.ps1" -VMHost $vmHost -LibraryResource $LibraryResource -RunAsAccount $RunAsAccount -ScriptCommandSetting $scriptSetting -RunAsynchronously -JobVariable vmsan| Out-Null
        while ($vmsan.status -eq "Running") {start-sleep 5}
        if ($vmsan.status -eq "Failed")
        {
            return "Failed at virtual SAN Switch stage"
        }
        Invoke-SCScriptCommand -Executable "%SystemRoot%\system32\WindowsPowerShell\v1.0\powershell.exe" -TimeoutSeconds 300 -CommandParameters $configarray.VMQIndirectionTableScript -VMHost $vmHost -LibraryResource $LibraryResource -RunAsAccount $RunAsAccount -ScriptCommandSetting $scriptSetting -RunAsynchronously -JobVariable vmq | Out-Null
        while ($vmq.status -eq "Running") {start-sleep 5}
        if ($vmq.status -eq "Failed")
        {
            return "Failed at vmq indirection stage"
        } 
    return "Success"   
    } # inlinescript
    return $statuscode  
}
```

# Custom Resource: Postdeploy.ps1

Postdeploy.ps1 is dependent on the presence of a tool called NVSPBIND which can be downloaded here [http://gallery.technet.microsoft.com/Hyper-V-Network-VSP-Bind-cf937850](http://gallery.technet.microsoft.com/Hyper-V-Network-VSP-Bind-cf937850){:target="_blank"} and should be placed in the custom resource as well.

Postdeploy.ps1 will run on all hosts and it will handle the following tasks:

* Check for the optical drive and assign it drive letter Z if one exists.
* Install the following windows features and management tools (note: the Hyper-V role and MPIO feature are already installed by the bare-metal host deployment job):
    * SNMP service
    * Hyper-V PowerShell module
    * Failover Clustering
* Run the HP Service Pack for ProLiant which installs missing drivers, updates firmware and installs management components.
* Check for visible SAN storage providers and add them to the MSDSM supported hardware list
* Set the Microsoft DSM to default to Round Robin load balancing.
* Enable ODX support.
* Reconfigures network adapter settings:
    * For the Live Migration Network adapter:
        * Disables DNS registration
    * For the Cluster Shared Volume adapter:
        * Disables DNS registration
* Change the network adapter binding order to MGMT first, CSV second and LM third.
* Disables use of LMHOSTS and NetBIOS over TCP/IP for al network adapters.
* Renames Management OS vNICs
* Enables Advanced Session Mode
* Disables HP Agent Cluster Monitoring
* Disables SMB Multichannel (this is done because all vNICs will report being 10Gbps NICs and we want to control CSV traffic to flow over the CSV vNIC. By disable SMB multichannel we force NetFT rule compliancy. For more info see: [http://blogs.technet.com/b/cedward/archive/2013/11/06/windows-server-2012-smb-multichannel-and-csv-redirected-traffic-caveats.aspx](http://blogs.technet.com/b/cedward/archive/2013/11/06/windows-server-2012-smb-multichannel-and-csv-redirected-traffic-caveats.aspx){:target="_blank"} )
* In this case this can be safely done, when using a SOFS cluster you would have to rethink the puzzle and probably will end up using SMB multichannel restrictions by using constraints.

```powershell
[string]$HPSUMShare = "\\fileserver\swpackages2014"
[string]$LocalHPSUMDir = $HPSUMShare.Split("\")[-1]
[string]$LMNetwork = "10.10.10.*"
[string]$CSVNetwork = "10.10.11.*"
[string]$MGMTNetwork = "10.10.9.*"
[int]$LMVlan = 20
[int]$CSVVlan = 10
[int]$MGMTVlan = 0
if (Get-CimInstance Win32_Volume -Filter 'drivetype = 5')
{
    Get-CimInstance Win32_Volume -Filter 'drivetype = 5' | Set-CimInstance -Arguments @{driveletter = “Z:”}
}
Install-WindowsFeature SNMP-Service,Hyper-V-PowerShell,Failover-Clustering -IncludeManagementTools

if (test-path -Path c:\setup)
{
    Remove-Item C:\setup -Force -Recurse
}
New-Item -Path c:\ -Name setup -ItemType directory
Copy-Item -Path  \\fileserver\swpackages2014 -Recurse -Destination C:\setup
start-process -FilePath "C:\setup\$LocalHPSUMDir\x64\hpsum_bin_x64.exe" -ArgumentList "/silent /use_latest /allow_non_bundle_components /allow_update_to_bundle /use_snmp" -wait -NoNewWindow
Remove-Item C:\setup -Force -Recurse
Get-MPIOAvailableHW | New-MSDSMSupportedHW
Set-MSDSMGlobalDefaultLoadBalancePolicy -Policy RR
Set-ItemProperty hklm:\system\currentcontrolset\control\filesystem -Name "FilterSupportedFeaturesMode" –Value 0

Get-NetAdapter | ?{$_.status -eq "up"} | ForEach-Object {

    if (($_ | Get-NetAdapterBinding -ComponentID ms_tcpip).enabled -eq $true)
    {
        if ($_ | Get-NetIPAddress |?{$_.ipaddress -like $LMNetwork})
        {
            $_ | Set-DNSClient -RegisterThisConnectionsAddress $false
            $script:LM = $_
        }
        
        elseif ($_ | Get-NetIPAddress |?{$_.ipaddress -like $CSVNetwork})
        {
            $_ | Set-DNSClient -RegisterThisConnectionsAddress $false
            $script:CSV = $_
        }
        
        elseif ($_ | Get-NetIPAddress |?{$_.ipaddress -like $MGMTNetwork})
        {
            $script:MGMT = $_
        }
    }
    else
    {
        # do nothing
    }
}
#binding order

if ($LM)
{
    .\nvspbind.exe /++ $LM.name ms_tcpip
}
if ($CSV)
{
    .\nvspbind.exe /++ $CSV.name ms_tcpip
}
.\nvspbind.exe /++ $MGMT.name ms_tcpip

#Disable Netbios and LMHOSTS
Invoke-CimMethod -ClassName Win32_NetworkAdapterConfiguration -MethodName EnableWINS -Arguments @{DNSEnabledForWINSResolution = $false; WINSEnableLMHostsLookup = $false}
Get-CimInstance win32_networkadapterconfiguration -filter 'IPEnabled=TRUE' | Invoke-CimMethod -MethodName settcpipnetbios -Arguments @{TcpipNetbiosOptions = 2}
#disable disconnected network adapters
Get-NetAdapter -Physical | ?{$_.status -eq "disconnected"} | Disable-NetAdapter -Confirm:$false
Get-VMNetworkAdapter -ManagementOS | %{

    if ((Get-VMNetworkAdapterVlan -VMNetworkAdapter $_).AccessVlanId -eq $LMVlan)
    {
        Rename-VMNetworkAdapter $_ -NewName "LM"
    }
    elseif ((Get-VMNetworkAdapterVlan -VMNetworkAdapter $_).AccessVlanId -eq $CSVVlan)
    {
        Rename-VMNetworkAdapter $_ -NewName "CSV"
    }
    elseif ((Get-VMNetworkAdapterVlan -VMNetworkAdapter $_).AccessVlanId -eq $MGMTVlan)
    {
        Rename-VMNetworkAdapter $_ -NewName "MGMT"
    }

}
Set-VMHost -EnableEnhancedSessionMode $true
#Disable HP Cluster Monitoring
$clientreg = Get-ItemProperty HKLM:\SYSTEM\CurrentControlSet\Services\CqMgHost\Parameters -Name SubServices
Set-ItemProperty HKLM:\SYSTEM\CurrentControlSet\Services\CqMgHost\Parameters -Name SubServices -Value ($clientreg.subservices | ?{$_ -ne "CPQCLUS"}) 
#Disable SMB Multichannel 
Set-SmbClientConfiguration -EnableMultiChannel $false -Confirm:$false
Set-SmbServerConfiguration -EnableMultiChannel $false -Confirm:$false 
```

# Custom Resource: BL460_converged.ps1

BL460_converged.ps1 will run on BL460 blades only and it will handle the following tasks:

* Install script prerequisite (if not already installed) Hyper-V PowerShell module.
* Capture Hyper-V management IP address.
* Add the physical management NIC as a team member to the Infra team.
* Add vNIC for new management connections
* Apply captured IP address to management vNIC and configure gateway and DNS settings.

```powershell
[string]$logicalswitchname = "infra"
[string]$MGMTNetwork = "10.10.9.*"
[string]$MGMTDefaultGateway = "10.10.9.1"
[string[]]$DNSServers = "10.10.0.2","10.10.0.3"
[string]$DNSSuffix = "lab.local"

Install-WindowsFeature Hyper-V-PowerShell
$ip = Get-NetIPAddress -IPAddress $MGMTNetwork
$NICToBeAdded = $ip.InterfaceAlias
Add-NetLbfoTeamMember -Name $NICToBeAdded -Team $logicalswitchname -Confirm:$false
start-sleep 5
Add-VMNetworkAdapter -ManagementOS -Name MGMT -SwitchName $logicalswitchname -Confirm:$false
$mgmtVnic = Get-NetAdapter -InterfaceDescription "Hyper-V Virtual Ethernet Adapter*"
New-NetIPAddress -InterfaceIndex $mgmtVnic.ifindex -IPAddress $ip.IPAddress -PrefixLength 24 -DefaultGateway $MGMTDefaultGateway -Confirm:$false
Set-DnsClientServerAddress -InterfaceIndex $mgmtVnic.ifIndex -ServerAddresses $DNSServers -Confirm:$false
Set-DnsClient -InterfaceIndex $mgmtVnic.ifIndex -ConnectionSpecificSuffix $DNSSuffix -RegisterThisConnectionsAddress $true -UseSuffixWhenRegistering $true -Confirm:$false
Register-DnsClient
```

# Custom Resource: BL460_npiv.ps1

This script requires the “OneCommand Fibre Channel and Converged Network Adapter Configuration Utility” to be installed (see HPSUM share pre-requisite).

BL460_npiv.ps1 will run on BL460 blades only and it will handle the following tasks:

* Enable NPIV driver support by:
    * Querying for a valid WWNP in the correct format
    * Supplying the previously acquired WWNP to an enable NPIV support command (this command will enable NPIV support globally).
* Set the HBA queue depth to 64 (default 32)
    * (Queue depth is the number of I/O requests (SCSI commands) that can be queued at one time on a storage controller. A higher number equals better performance but comes at a cost of maximum potential targets. Also, if set to high SAN controller port I/O depletion could occur resulting in worse performance).

```powershell
# enable NPIV support for FCOE HBA
[string]$wwnp = & 'C:\Program Files\emulex\Util\OCmanager\hbacmd.exe' listhbas local pt=fcoe | select-string "port wwn" | select -first 1
$wwnp = $wwnp.Substring(17)
& 'C:\Program Files\emulex\Util\OCmanager\hbacmd.exe' setdriverparam $wwnp G: P: enablenpiv 1
& 'C:\Program Files\emulex\Util\OCmanager\hbacmd.exe' setdriverparam $wwnp G: P: queuedepth 64 
```

# Custom Resource: BL460_vmsan.ps1

BL460_vmsan.ps1 will run on BL460 blades only and it will handle the following tasks:

* Query for Fibre Channel HBAs and create a VM San Switch for each one.

```powershell
[string]$VMSANName = "FCSAN"
if ((Get-VMSan).count -eq 0)
{
$i = 0
Get-InitiatorPort -ConnectionType FibreChannel | ForEach-Object{
        $i++
        New-VMSan -Name "$VMSANName$i" -HostBusAdapter $_
    }
}
else
{
    # do nothing
}
```

# Custom Resource: BL460_xxxx_vmq.ps1

BL460_xxxx_vmq.ps1 will run on BL460 blades only and it will handle the following tasks:

* Queries all physical network adapters which are up and
    * enable 9K Jumbo packet support for each
    * disable all checksum offloading support for each
* Enable Jumbo packets on the Live Migration and CSV vNICs.
* Configures the VMQ CPU indirection table by:
    * Assigning core 2 till 5 to the first NIC and core 6 till 9 to the second NIC which are joined to the Infra Team.
    * Assigning core 10 till 14 to the first NIC and core 15 till 19 to the second NIC which are joined to the VM Team.

The xxxx stands for an environment specific code like “Prod” or “Test”. In my environment I had different CPU types (Prod 10 cores / CPU and Test 8 cores /CPU).

```powershell
Get-NetAdapter -Physical | ?{$_.status -eq "up"}  | Get-NetAdapterAdvancedProperty -RegistryKeyword *jumbo* | Set-NetAdapterAdvancedProperty -RegistryValue 9014 -NoRestart
Get-NetIPAddress 10.10.10* | Get-NetAdapter| Get-NetAdapterAdvancedProperty -DisplayName "Jumbo Packet" | Set-NetAdapterAdvancedProperty -RegistryValue 9014 -NoRestart
Get-NetIPAddress 10.10.11* | Get-NetAdapter| Get-NetAdapterAdvancedProperty -DisplayName "Jumbo Packet" | Set-NetAdapterAdvancedProperty -RegistryValue 9014 -NoRestart
$infranic = (Get-NetLbfoTeamMember -Team infra).InterfaceDescription
Set-NetAdapterVmq -InterfaceDescription $infranic[0] -BaseProcessorNumber 2 -MaxProcessorNumber 5 -NoRestart
Set-NetAdapterVmq -InterfaceDescription $infranic[1] -BaseProcessorNumber 6 -MaxProcessorNumber 9 -NoRestart
$vmswitchnic = (Get-NetLbfoTeamMember -Team "vm switch").InterfaceDescription
Set-NetAdapterVmq -InterfaceDescription $vmswitchnic[0] -BaseProcessorNumber 10 -MaxProcessorNumber 14 -NoRestart
Set-NetAdapterVmq -InterfaceDescription $vmswitchnic[1] -BaseProcessorNumber 15 -MaxProcessorNumber 19 -NoRestart
Get-NetAdapter -Physical | ?{$_.Status -eq "up"} | Get-NetAdapterAdvancedProperty -RegistryKeyword *checksum* | Set-NetAdapterAdvancedProperty -DisplayValue disabled -NoRestart
Restart-NetAdapter -InterfaceDescription $infranic[0]
sleep 5
Restart-NetAdapter -InterfaceDescription $infranic[1]
sleep 5
Restart-NetAdapter -InterfaceDescription $vmswitchnic[0]
Restart-NetAdapter -InterfaceDescription $vmswitchnic[1]
```