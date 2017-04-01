---
title:  "SCVMM 2012 SP1: Remove Computer Tier from Running Service"
date:   2013-06-07 12:00:00
categories: ["SCVMM"]
tags: ["SCVMM"]
---
Ever needed to remove a tier from an already running service?

You would think you can just create an update from the service profile, delete the computer tier and update the service to the new version.
But this will not work, SCVMM will complain it cannot update the service because of a missing tier :(

What I did was to remove the VMs running in the Tier, go into SQL Management studio and run a little TSQL script.
Replace the TIERNAME and SERVICENAME placeholder (it's probably better to identify them based on their unique ID but I didn't have the time to look it up, will update this post later).

```sql
USE [VirtualManagerDB]
GO
delete from tbl_WLC_ComputerTier where name='TIERNAME'
delete from tbl_WLC_ComputerTierTemplate where name='TIERNAME'
update tbl_WLC_ServiceInstance
set ServiceStatus = 0
where name='SERVICENAME'
```

It will delete the Tier from the original template and service itself. It will also reset the status of the service to healthy again.
When done, update from an updates service template and your fine :)

HTH