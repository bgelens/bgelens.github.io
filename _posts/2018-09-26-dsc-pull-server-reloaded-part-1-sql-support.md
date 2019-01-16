---
title:  "DSC Pull Server reloaded. Part 1: SQL Support"
date:   2018-09-26 12:00:00
categories: ["Desired State Configuration","Pull Server"]
tags: ["Desired State Configuration","Pull Server"]
excerpt_separator: <!--more-->
---

Windows Server 1803 ([Semi-Annual Channel release](https://docs.microsoft.com/en-us/windows-server/get-started/semi-annual-channel-overview){:target="_blank"}) and the upcoming Windows Server 2019 introduces a big change for the DSC Pull Server in the form of SQL support. Having SQL as a (supported) backend unlocks a lot of potential for the "inbox" DSC Pull Server (making it more alive then ever). In this blog post, and the following blog posts in this series, I'm exploring some of the options that came to my mind and hopefully will get you inspired as well!

<!--more-->

Index:

* Part 1: SQL Support (this post)
* [Part 2: Managing the Pull Server with DSCPullServerAdmin](https://bgelens.nl/dsc-pull-server-reloaded-part-2-managing-the-pull-server-with-dscpullserveradmin)
* [Part 3: Pre-create the Pull Server database](https://bgelens.nl/dsc-pull-server-reloaded-part-3-precreate-pull-server-database)
* [Part 4: Migrate EDB or MDB to SQL](https://bgelens.nl/dsc-pull-server-reloaded-part-4-migrate-edb-mdb-to-sql)

# SQL Support

The main pain points for the "inbox" DSC Pull server have always been about scalability, availability and the lack of administrative capabilities. This all changes with the SQL backed pull server. Things like, fetching reports for a single or multiple nodes (with or without knowledge of the AgentId up front), figure out what nodes are under management of the pull server and options to change the configuration name server side have all become very easy. Also, creating a farm of Pull Servers is not possible without any hacks.

Let's look at the new possible DSC Pull Server (Farm) architecture:

![pullserver-arch](/images/2018-09/pullserver-arch.png)

As you can see in the picture above, we can now have a HA SQL Database (I tested on both SQL Server and Azure SQL), one or more Pull Servers targeting the same database and sharing configuration mofs, modules and registration keys using shared storage. Also drawn in the diagram is an Administrative interface with SQL. For now, just know it is possible to interact with SQL directly but keep reading the posts in the blog series as I will layer things on top of this, like a REST API, to enhance the accessibility and usability for Administrative purposes.

## Setting it up

For this post, I'll setup a single (and simple, and unsecure) Pull Server with a SQL Database using the following DSC configuration.

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
        SqlConnectionString          = 'Provider=SQLOLEDB.1;Server=sql.mshome.net;Database=DemoDSC;User ID=SA;Password=Welkom01;Initial Catalog=master;'
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

Note that the configuration makes use of [xPSDesiredStateConfiguration](https://github.com/PowerShell/xPSDesiredStateConfiguration){:target="_blank"}  version 8.4.0.0 which has the SQL support coded in. SQL support started in an earlier version of the module but had some bugs, so make sure you have version 8.4.0.0 or later.

The SQL connection string used:

* specifies a Provider of `SQLOLEDB.1`. You could also make use of the `SQLNCLI11` provider but this requires you to install the [SQL Server Native Client](https://docs.microsoft.com/en-us/sql/relational-databases/native-client/applications/installing-sql-server-native-client?view=sql-server-2017){:target="_blank"} as an additional component of the Pull Server.
* specifies the database name, the first time the Pull Server API is connected to, a database will be created automatically. Not specifying the name in the connection string will result in a database called `DSC`.
* uses an `Initial Catalog` as `master`. This is required for the Database to get created. You could create the database up front using my PowerShell module [DscPullServerAdmin](https://github.com/bgelens/DSCPullServerAdmin){:target="_blank"} (more on this in a later post) which would allow you to leave out the `Initial Catalog`.
* uses a SQL login (with plain text password, can be encrypted manually). You could also use integrated authentication.

I've setup 2 VMs on my local Hyper-V:

* sql.mshome.net (A Windows Server 1803 server with SQL Server 2017 installed with a default SQL instance name
* pull.mshome.net (A Windows Server 1803 server)

As I'm using the `Default Switch` on my box, both VMs have the `mshome.net` domain suffix.

So let's run this configuration.

```powershell
PullServerSQL
Start-DscConfiguration -Path .\PullServerSQL -Wait -Verbose -Force
```

![runconfig](/images/2018-09/runconfig.png)

Done! The pull server is setup. Let's quickly check the web.config settings.

```powershell
([xml](Get-Content -Path C:\inetpub\PSDSCPullServer\web.config)).configuration.appsettings.GetEnumerator()
```

![webconfig](/images/2018-09/webconfig.png)

As you can see, the connection string is stored in plain text. So if you need to have this encrypted, you need to take [additional steps](https://msdn.microsoft.com/en-us/library/dtkwfdky.aspx){:target="_blank"}.

The database is not created yet as the API has not been called in any way.

![nodb](/images/2018-09/nodb.png)

We can force the creation by just calling the api manually.

```powershell
Invoke-WebRequest -Uri http://localhost:8080/PSDSCPullServer.svc -UseBasicParsing
```

![dbcreated](/images/2018-09/dbcreated.png)

A database is created containing 3 tables:

* Devices (LCMv1, WMF4 clients are registered here)
* RegistrationData (LCMv2+, WMF5+ clients are registered here)
* StatusReport (LCMv2+, WMF5+ client reports are stored here)

Great! we now have a SQL backed pull server and nodes can start onboarding! Also scaling it out is as simple as running the same configuration on a second pull server and having a loadbalancer in front of them. Second and additional servers will connect to the same database and thus know about all the registered nodes. You just need to handle the content of `c:\pullserver` to be synced between all servers or have it coming from shared storage like storage spaces direct.

That is it for this post. To be continued!
