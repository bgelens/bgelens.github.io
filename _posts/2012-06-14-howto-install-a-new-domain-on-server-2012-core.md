---
title:  "Howto install a new domain on Server 2012 Core"
date:   2012-06-14 12:00:00
categories: ["Active Directory"]
tags: ["Windows Server","Active Directory","PowerShell"]
---
```powershell
<#
    Setting up a 2012 RC Domain Controller using Server Core and Powershell only :D  When done installing and configuring your new password.
    In CMD type start powershell and continue
    This post has been updated to include CIM cmdlets instead of WMI
    Diskpart command has been removed and replaced by CIM cmdlets
    Added CIM commands for enabling RDP
    Some typos have been removed
#>

#lower executionpolicy so you can run scripts locally
Set-ExecutionPolicy remotesigned

#rename computer and restart
Rename-Computer -NewName “DC” -Restart

#check for InterFace number (in my case there is only 1 nic assigned. The rest of the configuration uses the interfaceindex number gathered here)
get-netadapter -physical

#rename netadapter to something you can remember
get-netadapter -interfaceindex 12 | rename-netadapter -newname “Public”

#bind new ipv4 address to interface (you must understand cidr notation, prefixlength of 24 means 24 bits are masked for the network 255.255.255.0)
New-NetIPAddress -IPAddress 192.168.1.3 -defaultgateway 192.168.1.1 -prefixlength 24 -interfaceindex 12

#set dns server
Set-DnsClientServerAddress -InterfaceIndex 12 -ServerAddresses 127.0.0.1

#configure dns client settings
Set-DNSClient -InterfaceIndex 12 -ConnectionSpecificSuffix “lab.int” -RegisterThisConnectionsAddress $true -UseSuffixWhenRegistering $true

#disable LMHOST (system wide setting)
Invoke-CimMethod -ClassName Win32_NetworkAdapterConfiguration -MethodName EnableWINS -Arguments @{DNSEnabledForWINSResolution = $false; WINSEnableLMHostsLookup = $false}

#disable netbios over TCP/IP for specific adapter
Get-CimInstance win32_networkadapterconfiguration -Filter ‘servicename = “netvsc”‘ | Invoke-CimMethod -MethodName settcpipnetbios -Arguments @{TcpipNetbiosOptions = 2}

#rename filesystem volume
set-volume -driveletter c -newfilesystemlabel System

#assing driveletter Z to dvd drive
Get-CimInstance Win32_Volume -Filter 'drivetype = 5' | Set-CimInstance -Arguments @{driveletter = “Z:”}

#removing unneeded binairies (hardening!) #this step is best executed when done. however, i wanted to show how to get binairies back in place when needed.
Get-WindowsFeature | ?{$_.installstate -eq “Available”} | remove-windowsfeature -remove

#adding binairies needed for Active Directory
new-item c:\installdvd -ItemType directory
#which edition of windows am i running?
Get-WindowsEdition -Online # make a note
#get the right image index from the source file on the installation dvd
Get-WindowsImage -ImagePath z:\sources\install.wim
#mount the correct image
mount-windowsimage -imagepath z:\sources\install.wim -index 3 -path c:\installdvd -readonly
install-windowsfeature AD-Domain-Services,DNS -IncludeManagementTools -Source C:\installdvd\Windows\WinSxS  dismount-windowsimage -path c:\installdvd -discard

#deploy active directory (server 2008R2 level, most new functionality work with 2008R2 levels and some tools like Scom 2012 momadadmin.exe fail when functional level is set to 2012).

#get safe mode admin password
$safemodeadminpwd = read-host “Safe mode admin Password:” -AsSecureString
#install active directory (-force parameter makes sure you won’t get prompted for confirmation)
Install-ADDSForest -DomainName “lab.int” -DomainNetbiosName “LAB” -DomainMode Win2008R2 -ForestMode Win2008R2 -InstallDns -SafeModeAdministratorPassword $safemodeadminpwd -Force
# the system will now reboot

#rename the default first site name
Get-ADReplicationSite | Rename-ADObject -NewName “DataCenter”

#Add subnet to the datacenter site
New-ADReplicationSubnet -Name "192.168.1.0/24" -Site DataCenter

#Enable the recycle bin
Enable-ADOptionalFeature "Recycle Bin Feature" -Scope Forest -Target lab.int -confirm:$false

#configure dns server forwarder for the internet (it’s my lab ofcource!)
Set-DnsServerForwarder -IPAddress 192.168.1.1

#enable RDP
get-CimInstance "Win32_TerminalServiceSetting" -Namespace root\cimv2\terminalservices | Invoke-CimMethod -MethodName setallowtsconnections -Arguments @{AllowTSConnections = 1; ModifyFirewallException = 1}

#set RDP to only accept NLA
get-CimInstance "Win32_TSGeneralSetting" -Namespace root\cimv2\terminalservices -Filter 'TerminalName = "RDP-Tcp"' | Invoke-CimMethod -MethodName SetUserAuthenticationRequired -Arguments @{UserAuthenticationRequired = 1}
```