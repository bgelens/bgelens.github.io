---
title:  "DSC Pull Server reloaded. Part 4: Migrate EDB or MDB to SQL"
date:   2019-01-16 12:00:00
categories: ["Desired State Configuration","Pull Server"]
tags: ["Desired State Configuration","Pull Server"]
excerpt_separator: <!--more-->
---

In the last post, we have seen how to use pre-create the Pull Server Database as an Azure SQL Database. It's always nice to be able to start fresh but a lot of you already deployed Pull Servers based on WMF5+ backed by an EDB or MDB database and you want to move forward right? Now it's time to figure out how to "migrate" your existing Pull Servers to a new SQL backed Pull Server. The key here is to again use [**DSCPullServerAdmin**](https://github.com/bgelens/DSCPullServerAdmin) to make your life easy!

If you haven't already:

```powershell
Install-Module DSCPullServerAdmin -Verbose -Force -Scope CurrentUser
```

<!--more-->

Index:

* [Part 1: SQL Support](https://bgelens.nl/dsc-pull-server-reloaded-part-1-sql-support)
* [Part 2: Managing the Pull Server with DSCPullServerAdmin](https://bgelens.nl/dsc-pull-server-reloaded-part-2-managing-the-pull-server-with-dscpullserveradmin)
* [Part 3: Pre-create the Pull Server database](https://bgelens.nl/dsc-pull-server-reloaded-part-3-precreate-pull-server-database)
* Part 4: Migrate EDB or MDB to SQL (this post)
* [Part 5: Containers!](https://bgelens.nl/dsc-pull-server-reloaded-part-5)

# Migration process

In general the migration from EDB or MDB to SQL follows these steps:

1. Install one or more Pull Servers backed by a SQL Database (with multiple, have a load balancer in front off them).
2. Copy all configurations, checksums, DSC Resource modules to the new Pull Server(s).
3. Stop the EDB / MDB based Pull Server.
4. Copy data from the EDB / MDB database into the new SQL Database.
5. Update the DNS record pointing at the old Pull Server to point to the new Pull Server(s) / load balancer

It's good to know that LCMs only use the Registration Key during initial onboarding to a Pull Server and from there on use their AgentId to communicate with the Pull Server. So once an LCM is onboarded, there need to be a record for it in the Database with the corresponding AgentId. If this is the case, the LCM is free to communicate with the Pull Server. Knowing this, you could go in the SQL Database and create the records manually. You could also re-register all your LCMs with the new Pull Server or... You can use `DSCPullServerAdmin` (I'm using version 0.4.2 for this blog post) to pull data out of the old EDB / MDB database and inject it into the new SQL Database in a single function call!

>You could potentially move the other way around as well (SQL to EDB, or EDB to MDB, or... ). Since `DSCPullServerAdmin` version 0.3 EDB is fully supported (before only Get operations where available, but now all the other operations like new, set and remove are available as well) and since version 0.4 MDB support is added with full functionality as well!

For this blog post we'll handle the scenario EDB to SQL as it will be (I expect) the most common scenario. I'll also be targeting the same server to keep things simple.

> At PSConfAsia 2018 I spoke about this subject in detail so if you like to watch a video instead of reading, please check my video section!

## Existing Pull Server

The existing EDB Pull Server is configured using the following configuration:

```powershell
configuration PullServerEDB {
    Import-DscResource -ModuleName PSDesiredStateConfiguration -ModuleVersion 1.1
    Import-DscResource -ModuleName xPSDesiredStateConfiguration -ModuleVersion 8.4.0.0

    WindowsFeature dscservice {
        Name   = 'Dsc-Service'
        Ensure = 'Present'
    }

    File PullServerFiles {
        DestinationPath = 'c:\pullserver'
        Ensure = 'Present'
        Type = 'Directory'
        Force = $true
    }

    xDscWebService PSDSCPullServer {
        Ensure                       = 'Present'
        EndpointName                 = 'PSDSCPullServer'
        Port                         = 8080
        PhysicalPath                 = "$env:SystemDrive\inetpub\PSDSCPullServer"
        CertificateThumbPrint        = 'AllowUnencryptedTraffic'
        ModulePath                   = "c:\pullserver\Modules"
        ConfigurationPath            = "c:\pullserver\Configuration"
        DatabasePath                 = "c:\pullserver" 
        State                        = 'Started'
        RegistrationKeyPath          = "c:\pullserver"
        UseSecurityBestPractices     = $false
        DependsOn                    = '[File]PullServerFiles', '[WindowsFeature]dscservice'
    }

    File RegistrationKeyFile {
        Ensure          = 'Present'
        Type            = 'File'
        DestinationPath = "c:\pullserver\RegistrationKeys.txt"
        Contents        = 'cb30127b-4b66-4f83-b207-c4801fb05087'
        DependsOn       = '[File]PullServerFiles'
    }
}
```

Let's check the web.config for the pull server:

```powershell
([xml](Get-Content -Path C:\inetpub\PSDSCPullServer\web.config)).configuration.appsettings.GetEnumerator()
```

![edbwebconfig](/images/2019-01/edbwebconfig.png)

## Managed Node

I have one managed node which is currently onboarded and able to communicate with the EDB Pull Server.

The meta configuration looks like this:

```powershell
[dsclocalconfigurationmanager()]
configuration lcm {
    Settings {
        RefreshMode = 'Pull'
    }

    ConfigurationRepositoryWeb SQLPullWeb {
        ServerURL = 'http://pullserver:8080/PSDSCPullServer.svc'
        RegistrationKey = 'cb30127b-4b66-4f83-b207-c4801fb05087'
        AllowUnsecureConnection = $true
        ConfigurationNames = 'MySuperServer'
    }

    ReportServerWeb SQLPullWeb {
        ServerURL = 'http://pullserver:8080/PSDSCPullServer.svc'
        RegistrationKey = 'cb30127b-4b66-4f83-b207-c4801fb05087'
        AllowUnsecureConnection = $true
    }
}
```

![lcmconfig](/images/2019-01/lcmconfig.png)

And to see that the LCM is able to talk to the Pull Server:

![lcmpullok1](/images/2019-01/lcmpullok1.png)

## Get data from EDB

Back on the Pull Server, we can check the content of the database using `DSCPullServerAdmin` module.

```powershell
New-DSCPullServerAdminConnection -ESEFilePath C:\pullserver\Devices.edb
```

![edberror](/images/2019-01/edberror.png)

Ouch, this is one of the reasons we need to stop the Pull Server to handle the migration. EDB is only accessible by one process at a time (this is also the reason the EDB based Pull Server is not scalable!).

Let's stop the Pull Server and try again.

```powershell
iisreset /stop
New-DSCPullServerAdminConnection -ESEFilePath C:\pullserver\Devices.edb
Get-DSCPullServerAdminRegistration
```

![edbaccess](/images/2019-01/edbaccess.png)

Great we have access!

## Destroy EDB Pull Server

Normally you would not have to do this but in my case, since I'm deploying the SQL backed Pull Server on the same VM as the EDB, I need to destroy the current Pull Server.

```powershell
configuration PullServerCleanup {
    Import-DscResource -ModuleName PSDesiredStateConfiguration -ModuleVersion 1.1
    Import-DscResource -ModuleName xPSDesiredStateConfiguration -ModuleVersion 8.4.0.0

    xDscWebService PSDSCPullServer {
        Ensure                       = 'Absent'
        EndpointName                 = 'PSDSCPullServer'
        UseSecurityBestPractices     = $false
        CertificateThumbPrint        = 'AllowUnencryptedTraffic'
    }
}

PullServerCleanup

Start-DscConfiguration -Wait -Verbose .\PullServerCleanup -Force
Remove-WebAppPool -Name PSWS
```

## Create the new SQL backed Pull Server

On my server, SQL is installed locally already. I'm using the same configuration as in the [first part](https://bgelens.nl/dsc-pull-server-reloaded-part-1-sql-support) of this blog series:

```powershell
configuration PullServerSQL {
    Import-DscResource -ModuleName PSDesiredStateConfiguration -ModuleVersion 1.1
    Import-DscResource -ModuleName xPSDesiredStateConfiguration -ModuleVersion 8.4.0.0

    WindowsFeature dscservice {
        Name   = 'Dsc-Service'
        Ensure = 'Present'
    }

    File PullServerFiles {
        DestinationPath = 'c:\pullserver'
        Ensure = 'Present'
        Type = 'Directory'
        Force = $true
    }

    xDscWebService PSDSCPullServer {
        Ensure                       = 'Present'
        EndpointName                 = 'PSDSCPullServer'
        Port                         = 8080
        PhysicalPath                 = "$env:SystemDrive\inetpub\PSDSCPullServer"
        CertificateThumbPrint        = 'AllowUnencryptedTraffic'
        ModulePath                   = "c:\pullserver\Modules"
        ConfigurationPath            = "c:\pullserver\Configuration"
        State                        = 'Started'
        RegistrationKeyPath          = "c:\pullserver"
        UseSecurityBestPractices     = $false
        SqlProvider                  = $true
        SqlConnectionString          = 'Provider=SQLOLEDB.1;Server=localhost;Database=DemoDSC;User ID=SA;Password=Welkom01;Initial Catalog=master;'
        DependsOn                    = '[File]PullServerFiles', '[WindowsFeature]dscservice'
    }

    File RegistrationKeyFile {
        Ensure          = 'Present'
        Type            = 'File'
        DestinationPath = "c:\pullserver\RegistrationKeys.txt"
        Contents        = 'cb30127b-4b66-4f83-b207-c4801fb05087'
        DependsOn       = '[File]PullServerFiles'
    }
}
```

After setting it up, let's quickly validate it is working.

![setupsqlpull](/images/2019-01/setupsqlpull.png)

Great! If you need more details about this setup, please check the [first blog post](https://bgelens.nl/dsc-pull-server-reloaded-part-1-sql-support).

## See that Managed node is not able to communicate

On the managed node, let's run `Update-DscConfiguration -Wait -Verbose`.

![lcmfail](/images/2019-01/lcmfail.png)

We see this now results in a 404 error, meaning the Pull Server has no knowledge of this node. Time to fix this.

## Migration time

Let's create a connection with both the EDB and the new SQL database.

```powershell
$edb = New-DSCPullServerAdminConnection -ESEFilePath C:\pullserver\Devices.edb
$sql = New-DSCPullServerAdminConnection -SQLServer localhost -Database DemoDSC

# see impact
Copy-DSCPullServerAdminData -Connection1 $edb -Connection2 $sql -ObjectsToMigrate RegistrationData -WhatIf

# migrate data
Copy-DSCPullServerAdminData -Connection1 $edb -Connection2 $sql -ObjectsToMigrate RegistrationData

# get data from sql
Get-DSCPullServerAdminRegistration -Connection $sql
```

![migratereg](/images/2019-01/migratereg.png)

In the above example I've chosen to only migrate registration data. But you could easily migrate StatusReports and Devices as well by adding on to the `ObjectsToMigrate` parameter values.

> Note that `Copy-DSCPullServerAdminData` has Connection1 and Connection2 parameters. It used to be only possible to migrate from EDB to SQL but since version 0.3 and 0.4 it is also possible to migratie from SQL to EDB, EDB to EDB, MDB to SQL, SQL to MDB, EDB to MDB, SQL to SQL and so on! Whatever you fancy.

I used the same directory for Configuration and Module artifacts so no need to migrate those. Keep in mind if you are moving stuff to a new server or to a farm that you need to move these items as well (potentially to some shared location in case of an HA setup).

Let's check the LCM again.

![lcmsuccess](/images/2019-01/lcmsuccess.png)

Great! We now have out new Pull Server operational!

Finally let's move over the status reports:

```powershell
# see impact
Copy-DSCPullServerAdminData -Connection1 $edb -Connection2 $sql -ObjectsToMigrate StatusReports -WhatIf

# migrate data
Copy-DSCPullServerAdminData -Connection1 $edb -Connection2 $sql -ObjectsToMigrate StatusReports

# get data from sql
Get-DSCPullServerAdminStatusReport -Connection $sql
```

![statusreportmig](/images/2019-01/statusreportmig.png)

And see we could also use the Pull Server Rest API to get the data returned.

On the Managed node run:

```powershell
$agentId = (Get-DscLocalConfigurationManager).AgentId
$irmArgs = @{
    Uri = "http://pullserver:8080/PSDSCPullServer.svc/Nodes(AgentId = '$agentId')/Reports"
    Headers = @{
        Accept = 'application/json'
        ProtocolVersion = '2.0'
    }
    UseBasicParsing = $true
}

$reports = (Invoke-RestMethod @irmArgs).value
$reports[0]
```

![restapi](/images/2019-01/restapi.png)

As you can see, this still works.

That is it for this post. To be continued!
