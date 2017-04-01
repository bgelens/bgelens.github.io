---
title:  "SMA Database purge job and SQL AlwaysOn AG"
date:   2014-12-03 13:00:00
categories: ["SMA"]
tags: ["SMA","SQL"]
---
As part of the SMA installation, a SQL job gets created which runs a stored procedure to purge old SMA jobs from the database every 15 minutes by default. See [http://technet.microsoft.com/en-us/library/dn469633.aspx](http://technet.microsoft.com/en-us/library/dn469633.aspx){:target="_blank"} for more information.
When you use SQL AlwaysOn AG for your SMA database this job should be created on the “secondary” node as well to make sure it runs no matter which node is acting as Active.

To copy the job over you can simply right click the Job on the source server (Server which was Active during SMA installation) and select Script Job as -> Create to -> New Query Editor Window from the SQL management studio.

![](/images/2014-12/export-job.jpg)

This will generate TSQL code which can be run on the target SQL server which creates the Job for you.

Secondly we want to adjust the jobs (on both servers) with some logic to identify if the job runs on the Active node or not. When it’s not, the job should skip the stored procedure call because it will result in an error.

Navigate to the job steps and edit the only step which is available.
Modify the database connection to master or the job will generate an error on the “Passive” node no matter what extra tsql we put in.
Next modify the command. Replace it with the following code:

```sql
IF CONVERT(sysname, DATABASEPROPERTYEX('SMA','Updateability')) = 'READ_WRITE '
BEGIN
exec SMA.Purge.PurgeNextBatch
END
```

After this is done on both nodes, the job will only run the stored procedure when the database is in Read_Write mode (no more errors!)

![](/images/2014-12/sma-sql-job-alwayson.jpg)