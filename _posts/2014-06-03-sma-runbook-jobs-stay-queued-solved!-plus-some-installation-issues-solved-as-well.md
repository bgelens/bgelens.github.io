---
title:  "SMA Runbook jobs stay Queued SOLVED! Plus some installation issues SOLVED as well!"
date:   2014-06-03 12:00:00
categories: ["SMA"]
tags: ["SMA","PowerShell"]
---
Today I was configuring SMA (Service Management Automation) and was confronted with a strange issue. All runbooks stayed queued and never were assigned to the runbook worker. After some investigation I saw an error event in the SMA event log (Microsoft-ServiceManagementAutomation/Operational).

![](/images/2014-06/smaserviceerror.jpg)

The event itself is very explanatory!
I took all kinds of troubleshooting steps figuring it was a permissions issue (Security error was mentioned in the event right).
It just wouldn’t resolve. Just before I was about reinstalling the Runbook Server I tried my last resort and blocked GPO Inheritance, force updated GPO settings and restarted the runbook service. As I watched the eventlog for the error to reappear, instead an informational event was logged that the Runbook server was ready to accept jobs!

![](/images/2014-06/smaserviceready.jpg)

Now to figure out what GPO settings were actually killing my runbook functionality I went on with a process of elimination. Eventually I found that the GPO setting Turn on Script Execution for Windows PowerShell was to blame.

![](/images/2014-06/gposcriptexecution.jpg)

It was configured to allow all scripts. Setting the policy to remotesigned or undefined resolved the issue. Why this is happening? I don’t know. I guess the GPO setting is really badly implemented since it also conflicts with running any (out of box) best practice analyzer.

I also found that the SMA Runbook Server installer isn’t really font of CredSSP to be already configured by GPO. When this is the case, the installer will fail and perform a rollback of the setup. The setup isn’t capable of checking the current settings, even when they are already correctly configured (which was the case, I tripple checked). CredSSP will be enabled for both client and service and fresh credential delegation for credssp is enabled for wsman/*.domain as well. So when you install your runbook server, you better make sure these settings are not controlled by any policy.

Another Runbook server installation issue I was confronted with was that as part of the installation an update-help -force is run. When no internet connectivity is available on the box (probably isn’t right?) this will take forever and you will think the installation is stalled. To work around the issue, enable internet connectivity before you run setup or kill the powershell process which setup has spun up to kill the update (the installation will finish without error).

HTH! Ben