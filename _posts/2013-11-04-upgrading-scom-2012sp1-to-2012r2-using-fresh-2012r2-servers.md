---
title:  "Upgrading SCOM 2012SP1 to 2012R2 using fresh 2012R2 Servers"
date:   2013-11-04 12:00:00
categories: ["SCOM"]
tags: ["SCOM"]
---
At the moment I’m busy upgrading my costumers Private Cloud to 2012R2 versions of System Center and Server. A process introduced to the costumer as Ever-greening. In this blog post I’ll give you the steps to upgrade SCOM 2012SP1 to R2 in a scenario which also requires fresh 2012R2 server installs for the management servers.

For the moment TechNet or any other blog I have found does not give you the steps to include a refresh of the OS as well. Hellas it is not possible to “introduce” a new 2012R2 server into the SP1 environment (I’ve tried, the database did not get discovered).

It is possible to upgrade SCOM first and just do an in-place of the OS (I doubt if it is supported though). I’ve tested this out in my lab, the end result worked out well but didn’t feel quit right (a bit of a performance drop, though this could be .NET pre-compilation post OS upgrade [http://blogs.msdn.com/b/davidnotario/archive/2005/04/27/412838.aspx](http://blogs.msdn.com/b/davidnotario/archive/2005/04/27/412838.aspx)). In the end I decided it was preferred to have SCOM 2012 R2 on clean servers and started developing a runbook to get this done. This runbook assumes SCVMM is already upgraded to R2 (upcoming blog) and thus the integration with SCOM is broken when started (SCVMM 2012R2 cannot integrate with SCOM 2012SP1).

## The environment:
A simple environment monitoring about 100 agents with 2 management servers for redundancy.
All agents are targeted to the first management server.
First management server has RMS emulator role.
First management server has the reporting service installed.
Second management server has ACS installed. 
Both host the web console. 
The databases (including the SSRS database) are offloaded to a dedicated SQL 2012SP1CU6 cluster (this is why the first management server hosts the SSRS instance).

## The runbook:
* Disable subscriptions (Mail).
* Remove SCVMM 2012SP1 management packs.
* Assign (check) all agents to first management server.
* Disable ACS forwarding.
* Full Backup OperationsmanagerACS database.
* Upgrade ACS server on second management server (will upgrade the database).
* Uninstall ACS server leaving the database.
* Uninstall second management server.
* Delete second management server from SCOM MGMT view: administration => device management => management servers
* Cleanup database ETL table if necessary (when over 100.000 rows.
* Check [http://technet.microsoft.com/en-us/library/jj899852.aspx](http://technet.microsoft.com/en-us/library/jj899852.aspx) for the tsql scripts.
* Full Backup Operationsmanager and OperationsmanagerDW database.
* These backups will be the source for a rollback when necessary.
* Reinstall second management server with Server 2012R2.
* Install SCOM 2012R2 Prerequisites on second management server: ```Add-WindowsFeature NET-WCF-HTTP-Activation45,Web-Static-Content,Web-Default-Doc,Web-Dir-Browsing,Web-Http-Errors,Web-Http-Logging,Web-Request-Monitor,Web-Filtering,Web-Stat-Compression,Web-Mgmt-Console,Web-Metabase,Web-Asp-Net,Web-Windows-Auth –Source d:\sources\sxs``` The source parameter is necessary for .NET Framework 3.5 Features.
* Install Microsoft System CLR Types for SQL Server 2012 [http://go.microsoft.com/fwlink/?LinkID=239644](http://go.microsoft.com/fwlink/?LinkID=239644)
* Install report viewer [http://www.microsoft.com/en-us/download/details.aspx?id=35747](http://www.microsoft.com/en-us/download/details.aspx?id=35747)
* Backup SCOM reporting and reporting temp database (not really sure if necessary, will be rebuild later).
* Install SCOM 2012R2 Prerequisites on first management server (see earlier bullet SQL CLR and Report viewer links).
* Upgrade SCOM to 2012R2 on first management server (yes! on the 2012 server).
* Backup reporting encryption keys
This is done through Reporting Services Configuration Manager: [http://technet.microsoft.com/en-us/library/ms157275.aspx](http://technet.microsoft.com/en-us/library/ms157275.aspx).
* Install SCOM 2012R2 on second management server (all roles with exception of reporting).
* Move RMS emulator role to second management server. ```Set-SCOMRMSEmulator``` [http://technet.microsoft.com/en-us/library/hh918454(v=sc.10).aspx](http://technet.microsoft.com/en-us/library/hh918454(v=sc.10).aspx)
* Assign all agents to second management server.
* Uninstall SCOM on first management server.
* Install Server 2012R2 on first management server.
* Install SCOM 2012R2 Prerequisites on first management server (see earlier bullet for Roles and Features PowerShell line and SQL CLR and Report viewer links).
* Install SSRS on first management server. [http://technet.microsoft.com/en-us/library/ms143711.aspx](http://technet.microsoft.com/en-us/library/ms143711.aspx)
* Configure SSRS and connect the database.
* Import encryption keys.
* ResetRSS (SCOM support tools [http://marthijnvanrheenen.wordpress.com/2013/01/15/scom-2012-reset-reportserver/](http://marthijnvanrheenen.wordpress.com/2013/01/15/scom-2012-reset-reportserver/)).
* Install SCOM 2012R2 on first management server (all roles).
* Move RMS emulator role back.
* Assign agents to first management server.
* Install ACS on second management server.
* Install ACS report models [http://technet.microsoft.com/en-us/library/hh298613.aspx](http://technet.microsoft.com/en-us/library/hh298613.aspx).
* Enable mail subscription.
* Import SCVMM 2012R2 management packs.
* Setup SCVMM integration.
* Update Agents (this is done through the pending management node if Agents where originally deployed through SCOM, otherwise “Enjoy” your manual / SCCM / GPO / other based deployment upgrade process).
In my case the view blanked out (no agents where visible through the console). PowerShell however showed me all Agents which were pending ```Get-SCOMPendingManagement | Approve-SCOMPendingManagement``` will put them back into the console.
You could also run ```Get-SCOMPendingManagement | Approve-SCOMPendingManagement -ActionAccount (Get-Credential)``` to immediately go on with the agent upgrade if you can use the same credentials for all (or most) agents.
* Enable ACS forwarding.
* Update management packs.
* Recreate scheduled backup of non-sealed management packs (I dump my overrides daily to XML :) )

Final tip, have patience, don’t rush this through!!! and you’ll end up with a lean, mean and mostly clean R2 environment.

Have Fun and Hope this Helps!