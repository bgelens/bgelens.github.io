---
title:  "Use SMA Complex Type asset to store product keys in encrypted form"
date:   2014-12-19 12:00:00
categories: ["SMA","WAPACK","Windows Azure Pack"]
tags: ["SMA","WAPACK","Windows Azure Pack","PowerShell"]
---
I’m working on a project were we install SQL on VMs via an SMA runbook. The runbook generates a configuration.ini and then kicks of a scripted install. We support SQL 2008R2 Standard and Enterprise and SQL 2012 Standard and Enterprise.

Before this project started, the customer had multiple install sources to support all 4 installation types. This was due to the fact they used the defaultsetup.ini to hardcode which SQL edition was to be installed via the PID property. The sources were available on a share which was available to half the company, so half the company had access to the Product Keys as well. I don’t think this is a very good practice.

To circumvent duplicate install files per SQL version and to better secure the product keys, I decided to add them to the SMA database as an SMA complex type asset.

What is an SMA complex type asset?

Well, first of all it cannot be added using the Azure Pack Admin site. You can add it via the PowerShell cmdlet Set-SMAVariable. The complex type can store hash arrays and other objects. I found out about the complex type via the Building Clouds blog here: [http://blogs.technet.com/b/privatecloud/archive/2014/09/29/automation-service-management-automation-amp-azure-automation-complex-asset-creation-and-usage.aspx](http://blogs.technet.com/b/privatecloud/archive/2014/09/29/automation-service-management-automation-amp-azure-automation-complex-asset-creation-and-usage.aspx){:target="_blank"} 

So now you know what a complex type is, here is how I utilized it to store the SQL Product Keys in one SMA asset as encrypted values. Note that Encrypted values in this context mean, the values are encrypted in the SMA database but can be accessed by runbooks directly. Therefore every SMA Admin (there is no other role at the moment) can access the Keys.

First off, I create a hash array like so:

```powershell
$keys = @{
    '2008R2STD' = 'VVVVV-WWWWW-XXXXX-YYYYY-ZZZZZ'
    '2008R2ENT' = 'VVVVV-WWWWW-XXXXX-YYYYY-ZZZZZ'
    '2012STD'   = 'VVVVV-WWWWW-XXXXX-YYYYY-ZZZZZ'
    '2012ENT'   = 'VVVVV-WWWWW-XXXXX-YYYYY-ZZZZZ'
} 
```

Next I add it to the SMA database as an encrypted complex type.

```powershell
Set-SmaVariable -Name SQLKeys -Value $keys -WebServiceEndpoint https://localhost -Encrypted
```

When you access the variable using Get-SMAVariable, you will not get the values back since they are encrypted.

![](/images/2014-12/complextype1.jpg)

When accessed through Azure Pack Admin site:

![](/images/2014-12/complextype2.jpg)

If you like you can convert the hash array to a PS custom object before committing it to the database, but I don’t see any advantages in doing this right now. The reason the type is unknown is because of the encryption. When the encryption would be turned off, you would see it was recognized as a complex type.

![](/images/2014-12/complextype3.jpg)

Now that this complex type is committed, you can use a runbook to access its content.

![](/images/2014-12/complextype4.jpg)

And as an example on how to use it in an actual runbook:

![](/images/2014-12/complextype5.jpg)

Good stuff!