---
title:  "SCVMM 2012 SP1, move host group assigned service to a cloud!"
date:   2013-05-29 12:00:00
categories: ["SCVMM"]
tags: ["SCVMM","PowerShell"]
---
I've been working witch SCVMM2012 SP1 for a couple of months now and I can tell you the product isn't pretty. Every day I tremble over a new found bug or shortcoming.
I will write about these bugs/shortcomings and tell about these and of course the workarounds we have come up with in my next couple of blog posts.

For today I've come up with a little PowerShell script that generates the necessary TSQL queries to assign an already deployed service and its VMs to a cloud.
You will probably want to do this when services are deployed to host groups when "the CLOUD" didn't existed yet (forgot to use the create cloud menu :) )
The GUI and PowerShell will tell you to redeploy the service when you try to assign the CLOUD to the VMs. When you utilize services for rapid deployment and as monitoring containers this will not be a valid option. So VM cloud assignment has been made impossible once the service is deployed.

I've provisioned some test services in a cloud and compared the database entries to the ones which were provisioned against the host group. I've noticed just one field that was different in both the host and service tables.
Once modified and the scvmmservice service has been restarted, the service and its VMs are now appearing in the cloud that was selected for them.
The script is dirty but does a terrific job :) Just copy the output and past it into the query section of SQL management studio (connected to the instance that is hosting the VMM database off course), validate and execute. Restart the scvmmservice service and you’re done.

The script will give you a grid view twice to select the cloud you want to assign the service to and to select the service which needs to be assigned. HTH! Greets

```powershell
Import-Module virtualmachinemanager
$cloud = (Get-SCCloud | Out-GridView -PassThru).id.tostring()
$service = Get-SCService -All |  Out-GridView -PassThru
$servicename = $service.name.ToString()
$servicevms = Get-SCVirtualMachine -Service $service
[string]$vmupdate = ""
[string]$vmupdate += $servicevms | %{$id = $_.id; "objectid='$id'`n or"}
$vmupdate = "where " + $vmupdate.TrimEnd("`n or") + ";"
$tsql = @"
USE [VirtualManagerDB]
GO
--move service to Cloud
update [dbo].[tbl_WLC_ServiceInstance]
set CloudId = '$cloud',
HostGroupId = NULL
where name='$servicename'
--move service VMs to cloud
update [dbo].[tbl_WLC_VObject]
set CloudId = '$cloud'
$vmupdate
"@
$tsql
```

## Update01: 
it appears there are some overlooked interdependencies. I was recycling a computer tier today by deleting a server and scaling the tier out again which wouldn't allow me to proceed. I actually had to delete the cloud assignment again to fulfill the task. Will look into this if and when I have the time.

## Update02: fixed it :)
It appears I forgot to clear out a table column indicating the service has been deployed to the hostgroup. The service actually was assigned to both the Cloud and the host group which made it impossible to apply changes. Now it's running just fine and taking my updates!