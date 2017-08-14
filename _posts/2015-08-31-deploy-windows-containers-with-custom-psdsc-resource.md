---
title:  "Deploy Windows Containers with custom PSDSC resource"
date:   2015-08-31 12:00:00
categories: ["Desired State Configuration","PSDSC","Container"]
tags: ["PowerShell","PSDSC","Desired State Configuration","Container"]
---

This blog post is written and published on Hyper-V.nu: [https://hyper-v.nu/archives/bgelens/2015/08/deploy-windows-containers-with-custom-psdsc-resource/](https://hyper-v.nu/archives/bgelens/2015/08/deploy-windows-containers-with-custom-psdsc-resource/){:target="_blank"}

Last Sunday I was going through the MVA course [What’s New in Windows Server 2016 Preview](https://www.microsoftvirtualacademy.com/en-US/training-courses/what-s-new-in-windows-server-2016-preview-12592){:target="_blank"}. Eventually I ended up at the Windows Containers module and figured it would be nice if I had a Desired State Configuration resource to declaratively deploy containers with, including their initialization script. When I thought about this a bit more, I figured this could result in container automation similar to the dockerfile concept but then in PowerShell DSC configuration DSL. The big benefits being abstraction of lower level configuration and source controlled container recipes!

Since the Windows container tech is fresh of the boat, making its first appearance in [Windows Server TP3](https://www.microsoft.com/en-us/evalcenter/evaluate-windows-server-technical-preview){:target="_blank"}, don’t expect anything in the Desired State Configuration area released by Microsoft yet. Luckily the Containers PowerShell module and the container specific extensions made to Invoke-Command and Enter-PSSession are working fine. They provide a solid basis to create a custom DSC resource on top off (read more about Container management using PowerShell [here](https://msdn.microsoft.com/en-us/virtualization/windowscontainers/quick_start/manage_powershell){:target="_blank"}).

If you haven’t started playing with containers yet, start [here](https://msdn.microsoft.com/en-us/virtualization/windowscontainers/quick_start/container_setup){:target="_blank"}.

I started creating a DSC resource module and put in the bare minimum to make it functional. You can find, and contribute if you like, the project on my GitHub [here](https://github.com/bgelens/cWindowsContainer){:target="_blank"}. If you just want to use it, hit the download ZIP file button, unblock the ZIP and expand it to c:\Program Files\WindowsPowerShell\Modules.

A configuration based on this DSC resource can look like this (this one works perfectly):

```powershell
configuration ContainerNginX {
    param (
        [String] $StartupScript
    )
    Import-DscResource -ModuleName cWindowsContainer

    cWindowsContainer NginX {
        Ensure = 'Present'
        Name = 'NginX'
        StartUpScript = $StartupScript
        SwitchName = 'Virtual Switch'
    }
}

$script = @'
Invoke-WebRequest -Uri 'http://nginx.org/download/nginx-1.9.4.zip' -OutFile 'c:\nginx-1.9.4.zip'
Unblock-File -Path 'c:\nginx-1.9.4.zip'
Expand-Archive -Path 'c:\nginx-1.9.4.zip' -DestinationPath C:\ -Force
Set-Location -Path C:\nginx-1.9.4
Start-Process nginx
'@

ContainerNginX -StartupScript $script
Start-DscConfiguration .\ContainerNginX -Wait -Verbose -Force
```

Stuff I know I still need to implement (but not limited to):

* Create Image from deployed container and create new containers from that
* Configure Host NAT rules

When I feel I have a solid enough module created, I’ll publish it to the PS Gallery.

Results:

![](/images/2015-08/BG_ContainersPSDSC_001.png)

What you see here is the configuration being loaded into memory. Then the MOF file is generated and applied. As a result, the container is running and configured.

![](/images/2015-08/BG_ContainersPSDSC_003.png)

Get-DscConfiguration shows the defined settings.

![](/images/2015-08/BG_ContainersPSDSC_002.png)

When we look in the container, we see the nginx process running. Now we need to create a firewall rule and a PAT rule on the host to allow traffic to pass to the container on a specific port. For now, I’ll leave this manually until I figured out a smart way to do this.

```powershell
New-NetFirewallRule -Name Port80 -DisplayName Port80 -Enabled True -Direction Inbound -LocalPort 80 -Action Allow -Protocol tcp
Add-NetNatStaticMapping -NatName "ContainerNat" -Protocol TCP -ExternalIPAddress 0.0.0.0 -InternalIPAddress 172.16.0.2 -InternalPort 80 -ExternalPort 80
```

When a browser is targeted at the host ip address it should now open the nginx test page.

![](/images/2015-08/BG_ContainersPSDSC_004.png)

You might wonder why I did not configure the container runtime with DSC. Turns out, the WinRM service cannot be started (known issue, see here) which makes it impossible for the Local Configuration Manager (LCM) to apply a configuration.

Have fun with Containers!