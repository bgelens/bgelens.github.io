---
title:  "DSC Pull Server reloaded. Part 3: Pre-create the Pull Server Database"
date:   2018-11-01 12:00:00
categories: ["Desired State Configuration","Pull Server"]
tags: ["Desired State Configuration","Pull Server"]
excerpt_separator: <!--more-->
---

In the last post, we have seen how to use some of the functions of [DSCPullServerAdmin](https://www.powershellgallery.com/packages/DSCPullServerAdmin){:target="_blank"} to make your life easier! In this post, we'll look at another function from this module which allows you to pre-create the Pull Server Database. This is nice if you have a SQL Server where you don't have the correct permissions to setup new databases (but are allowed to connect to a database of course) and somebody else needs to create it, or where you are trying to host the Database in a service like Azure SQL where the Pull Server itself is not capable of creating the Database due the inability to use the initial catalog in the connection string.

In this post we'll use Azure SQL as our target Database platform.

<!--more-->

Index:

* [Part 1: SQL Support](https://bgelens.nl/dsc-pull-server-reloaded-part-1-sql-support)
* [Part 2: Managing the Pull Server with DSCPullServerAdmin](https://bgelens.nl/dsc-pull-server-reloaded-part-2-managing-the-pull-server-with-dscpullserveradmin)
* Part 3: Pre-create the Pull Server database (this post)

# Creating Azure SQL DSC Pull Server Database

So let's create an Azure SQL Server and Database. I'm using the `AZ` PowerShell Modules (version 0.4.0 at the time of writing).

```powershell
# create a new ResourceGroup
$rg = New-AzResourceGroup -Name pullsrv -Location 'West Europe'

# prompt for the SQL login credentials
$sqlCred = Get-Credential -Message 'Provide SQL login Credentials'

# create a new Azure SQL Server
$sqlServer = $rg | New-AzSqlServer -ServerName pullsrv -SqlAdministratorCredentials $sqlCred -Location $rg.Location

# add your public ip address to the Azure SQL firewall to allow connections and allow all connections from Azure ips
$myPublicIp = (Invoke-RestMethod -Uri http://ipinfo.io/json -UseBasicParsing).ip
$null = $sqlServer | New-AzSqlServerFirewallRule -FirewallRuleName home -StartIpAddress $myPublicIp -EndIpAddress $myPublicIp 
$null = $sqlServer | New-AzSqlServerFirewallRule -AllowAllAzureIPs

# use DSCPullServerAdmin module to pre-create the database
New-DSCPullServerAdminSQLDatabase -SQLServer $sqlServer.FullyQualifiedDomainName -Credential $sqlCred -Name pullsrvdb -Confirm:$false

# print out the connection string
'Provider=SQLOLEDB.1;Server={0};Database={1};User ID={2};Password={3};' -f @(
    $sqlServer.FullyQualifiedDomainName,
    'pullsrvdb',
    $sqlCred.UserName,
    $sqlCred.GetNetworkCredential().Password
)
```

For me the above script generated the connection string:

```
Provider=SQLOLEDB.1;Server=pullsrv.database.windows.net;Database=pullsrvdb;User ID=myUserName;Password=myPassword;
```

# Create a new Pull Server backed by Azure SQL Database

Now I have setup the Database in Azure we can install the Pull Server.

```powershell
configuration PullServerSQL {
    param (
        [Parameter(Mandatory)]
        [ValidateNotNullOrEmpty()]
        [string] $ConnectionString
    )
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
        SqlConnectionString          = $ConnectionString
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

$connectionString = 'Provider=SQLOLEDB.1;Server=;Database=pullserverdb;User ID=myUserName;Password=myPassword;'

PullServerSQL -ConnectionString $connectionString
Start-DscConfiguration -Path .\PullServerSQL -Wait -Verbose -Force
```

To test if the Pull Server is working properly let's try to communicate with it.

```powershell
(Invoke-WebRequest -Uri http://localhost:8080/PSDSCPullServer.svc -UseBasicParsing).StatusCode
```

This should result in a statuscode of `200`.

So we setup a Pull Server with Azure SQL. Let's onboard a node.

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
    }

    ReportServerWeb SQLPullWeb {
        ServerURL = 'http://pullserver:8080/PSDSCPullServer.svc'
        RegistrationKey = 'cb30127b-4b66-4f83-b207-c4801fb05087'
        AllowUnsecureConnection = $true
    }
}

lcm
Set-DscLocalConfigurationManager .\lcm -Verbose
```

Let's take a look in the Azure Portal to see if an entry was made. Open the Query editor in the Database blade and login.

![login](/images/2018-11/login.png)

Once logged in, type the following query and run it.

```sql
select * from RegistrationData
```

![query](/images/2018-11/query.png)

Great! And of course we can make use of DSCPullServerAdmin to connect to the Database as well.

```powershell
New-DSCPullServerAdminConnection -SQLServer pullsrv.database.windows.net -Database pullsrvdb -Credential pullsrv
```

![loginps](/images/2018-11/loginps.png)

```powershell
Get-DSCPullServerAdminRegistration
```

![registrationps](/images/2018-11/registrationps.png)

```powershell
Get-DSCPullServerAdminStatusReport -Top 1
```

![reportps](/images/2018-11/reportps.png)

That is it for this post. To be continued!
