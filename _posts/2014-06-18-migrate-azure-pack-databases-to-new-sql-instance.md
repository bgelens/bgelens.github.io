---
title:  "Migrate Azure Pack databases to new SQL Instance"
date:   2014-06-18 12:00:00
categories: ["WAPACK","Windows Azure Pack"]
tags: ["WAPACK","PowerShell","SQL","Windows Azure Pack"]
---
In my environment I have a very simple Azure Pack implementation. It only serves as an administration point for SMA so I have set it up very minimalistic.
My environment exists out of the following components:

* WindowsAuthSite
* AdminAPI
* AdminSite
* TenantAPI

The TenantAPI is mandatory, otherwise the AdminSite won’t function.

In my case I thus have only 3 databases:

* Microsoft.MgmtSvc.Config
* Microsoft.MgmtSvc.PortalConfigStore
* Microsoft.MgmtSvc.Store

I will describe here how to move these database to a new SQL instance (like with my previous post about the SMA database). This procedure should also work for fully implemented Azure Pack environments, including distributed setups (you just have to visit some extra servers :) ).

The Azure Pack database migration is a lot simpler than the SMA database migration since the only database referenced I have found existed in the connection strings of the web.config files.

First you stop all Azure Pack web services. Then you move over the databases and logins + SQL Accounts (see my previous post about SMA database migration). The “old” databases will be configured offline for the time being and detached / deleted later on.

Then the real challenge begins :)

The web.config files are encrypted so they should be decrypted first. You can do this through the following PowerShell code:

```powershell
Get-MgmtSvcNamespace | %{
    Unprotect-MgmtSvcConfiguration -Namespace $_
}
```

This code will unencrypt all web.config files associated with the namespaces installed on the local server. When this has run you can go to IIS manager and reconfigure the connection strings.

![](/images/2014-06/wapdatabase1.jpg)

(Side note) When you screwed up your SQL accounts associated with Azure Pack or you need to do something else with them, here you can find there passwords!
When every connection string is adjusted, you can run the following PowerShell code to encrypt the web.config files again:

```powershell
Get-MgmtSvcNamespace | %{
    Protect-MgmtSvcConfiguration -Namespace $_
} 
```

Start the web sites and you’re done!

HTH, Ben