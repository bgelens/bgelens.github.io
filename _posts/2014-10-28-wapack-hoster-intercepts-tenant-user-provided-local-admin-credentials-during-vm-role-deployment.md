---
title:  "WAPack hoster intercepts tenant user provided local admin credentials during VM role deployment (fixed in UR4)"
date:   2014-10-28 12:00:00
categories: ["WAPACK","Windows Azure Pack"]
tags: ["WAPACK","SMA","Windows Azure Pack","SCVMM","SPF","SQL"]
---
I started this proof of concept because I was testing the integration capabilities of SMA with WAPack. During my initial investigation I found this [blog post](http://www.miru.ch/linking-sma-runbooks-to-azure-pack-vm-cloud-events-and-get-job-parameters/){:target="_blank"} by Michael Rueefli which saved me a lot of work.

The goal at first was to get more information from the tenant user during VM role deployment and make that information actionable. For example, the tenant users checks a box for an extra service

I managed to expand the VM role resource definition file with some parameters which where unassigned and exposed them in the view definition sections. After publishing and assigning the VM role I tried to trap the extra information with an SMA runbook which proofed to be very simple. All parameter declaration data is send to the runbook as the $Params object.

Besides the information I was looking for I also saw the local administrator credentials which are provided by the user during VM role deployment. Even though the parameter type of credential is used, the username and password is send in clear text. This really surprised me which is why I eventually wrote this blog post.

Thankfully communication between WAPack, SPF and SMA is secured on the wire using SSL/TLS which makes sure the password can’t be snooped of the wire (easily). What could be kind of a big deal though is the fact that a WAPack hoster could easily intercept the credentials typed by the tenant user who is **unaware** of this. I noticed the same holds true for parameters of the type secure string.

Note that the data send to the $Resourceobject parameter holds the same data but the actual passwords and secure strings are replaced with “\_\_\*\*\_\_” which of course is a good thing!

I’ve created a simple proof of concept (see below) which stores the local administrator username and password including a value from a secure string in a simple database. I used the Domain Controller example (unmodified) downloaded with the WebPI utility to illustrate.

Before I decided to post this information to my blog, I decided together with my colleague Hans Vredevoort to contact Microsoft about this finding. Microsoft investigated and reported the behavior was due to a bug which apparently had resurfaced as it was patched before. I decided to postpone posting my findings until it was fixed. Now I can safely post this information because Microsoft just fixed this behavior in UR4. For the release notes see: [http://support2.microsoft.com/kb/2992021](http://support2.microsoft.com/kb/2992021){:target="_blank"}.

## Create the Passwords Database

Fist lets create a simple database in SQL called passwords. Then add a table to it called PWD. Add the columns as illustrated in the screenshot and save the table. I added the id column and enabled the identity specification so SQL will automatically assigns an integer, this is however not necessary for this proof of concept.

![](/images/2014-10/table.jpg)

You can use the following TSQL script to create the table:

```sql
USE [passwords]
GO
/****** Object:  Table [dbo].[PWD]    Script Date: 9-9-2014 16:34:18 ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[PWD](
    [id] [int] IDENTITY(1,1) NOT NULL,
    [UserRoleID] [text] NOT NULL,
    [Username] [text] NULL,
    [Password] [text] NULL,
    [Computername] [text] NOT NULL,
    [Owner] [text] NOT NULL,
    [SecureString] [text] NULL,
 CONSTRAINT [PK_PWD] PRIMARY KEY CLUSTERED
(
    [id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY] TEXTIMAGE_ON [PRIMARY]
 
GO
```

Add the SMA Runbook Service Service Account to the SQL logins and grant it DBO rights over the passwords database by running the following TSQL script (don’t forget to change the domain and samaccountname of the service account):

```sql
USE [master]
GO
CREATE LOGIN [Domain\SASMARUN] FROM WINDOWS
GO
 
USE [passwords]
GO
CREATE USER [Domain\SASMARUN] FOR LOGIN [Domain\SASMARUN] WITH DEFAULT_SCHEMA=[dbo]
GO
```

## Create the Runbook

Import the following runbook into SMA. Replace the variable data at the start of the workflow to reflect your own environment and add the SPF tag to the runbook so a trigger can be configured for it.

The runbook depends on a credential asset ‘SCVMM Service Account’ and string variable ‘VMMServer’ to be present to connect to SCVMM. I discovered when using ## symbols with the computername (which you should for scaling purposes), the actual computer names are not reported back. This is why the runbook connects to SCVMM and looks for them.

```powershell
workflow Store-LocalAdminCredInDatabase
{
    param
    (
        [object]$resourceObject,
        [object]$params
    )
    #change the following variables so they match your environment
    $ServerInstance = "sql.domain.int\cloudos"
    $Database = "passwords"
    $tableName = "pwd"
    $vmm = Get-AutomationVariable -Name 'VMMServer'
    $vmmcred = Get-AutomationPSCredential -Name 'SCVMM Service Account'
 
    # with thanks to Michael Rueefli who did this part on his blog post before me
    # I adapted his solution so it can handle the Admin:Password combinations as well.
    $toexpend = $params.ResourceConfiguration
    $toexpend = $toexpend.ParameterValues
    $toexpend = $toexpend -split (',') -replace ('"','') -replace ('{','') -replace ('}','')
    $output = inlinescript {
        $paramhashtable = @{}
        Foreach ($p in $using:toexpend)
        {
            $value = ""
            $key = ($p -split (':'))[0]
            #check if key has multiple values
            if ($p -split (':').count -gt 2)
            {
                #if multiple values exist, append them with : as a separator
                $p -split (':') | ?{$_ -ne $key} | %{
                    $value += $_ + ":"
                }
                $value = $value.trimend(":")
            }
            else
            {
                $value = ($p -split (':'))[1]
            }
            $paramhashtable.Add($key,$value)
        }
        return $paramhashtable
    }
    $output
 
    #Gather computernames from SCVMM
    $vmnames = inlinescript {
        import-module virtualmachinemanager
        (get-cloudresource -id $using:resourceobject.id).vms.name
    } -pscomputername $vmm -PSCredential $vmmcred
 
    inlinescript {
        $ConnectionTimeout = 15
        $QueryTimeout = 600
        $BatchSize = 50000
        $conn=new-object System.Data.SqlClient.SQLConnection
        $ConnectionString = "Server={0};Database={1};Integrated Security=True;Connect Timeout={2}" -f $using:ServerInstance,$using:Database,$ConnectionTimeout
        $conn.ConnectionString=$ConnectionString
        $conn.Open()
        $VMRoleAdminCredential = $using:output.VMRoleAdminCredential
        foreach ($VM in $using:vmnames)
        {
            $query = "INSERT INTO dbo.$using:tableName (userroleid,username,password,ComputerName,Owner,SecureString) VALUES ('$($using:resourceObject.UserRoleID)','$($VMRoleAdminCredential.split     (":")[0])','$($VMRoleAdminCredential.split(":")[1])','$VM','$($using:resourceobject.owner)','$($using:output.DomainControllerWindows2012SafeModeAdminPassword)')"
            Write-Output "adding row to database using query"
            $query
            $command = New-Object -TypeName System.Data.SqlClient.SqlCommand
            $command.Connection = $conn
            $command.CommandText = $query
            $command.ExecuteNonQuery() | out-null
        }
        $conn.Close()
    }
}
```

Add SPF as a tag to the runbook.

![](/images/2014-10/spftag.jpg)

A trigger can now be added by going to VM Clouds -> Automation. In this case we target the VMRole object with action create.

![](/images/2014-10/vmrolecreatetrigger.jpg)

## Results

I’ll assume you have implemented the Domain Controller resource and made it available through a plan for the tenant user.

The tenant user will deploy a Domain Controller VM role through the gallery which triggers the runbook. The runbook will parse the data, check for the correct computer names and add the data to the database.

![](/images/2014-10/runbookoutput.jpg)

![](/images/2014-10/tableresult.jpg)

As you can see this implementation is very simple and only mend to show how easy it is to intercept supposedly secure data (until UR4).

Note that another runbook should be built for scaling-up and down actions because very little data is provided to SMA when this happens.

![](/images/2014-10/vmroletenantview.jpg)

Another example is the 2012R2 workgroup gallery item which I also did not modify. In this case I deployed 2 instances during initial deployment.

![](/images/2014-10/vmroletenantview2.jpg)

![](/images/2014-10/runbookoutput2.jpg)

![](/images/2014-10/tableresult2.jpg)

### Update November 9th, 2014.
Today I upgraded my test environment to UR4 and indeed verified the fix is implemented by removing the values from the Params object.
However, secure strings are still send in the Params object in the clear. I send an e-mail to Microsoft to notify them the issue is fixed only partially.

![](/images/2014-10/securestring.jpg)