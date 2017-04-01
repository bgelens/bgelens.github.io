---
title:  "Integrating VM Role with Desired State Configuration Part 4 – Pull Server"
date:   2015-02-11 12:00:00
categories: ["WAPACK","Windows Azure Pack","Desired State Configuration","PSDSC"]
tags: ["WAPACK","Windows Azure Pack","PowerShell","VM Role","PKI","PSDSC","Desired State Configuration"]
---

This blog post is written and published on Hyper-V.nu: [http://hyper-v.nu/archives/bgelens/2015/02/integrating-vm-role-with-desired-state-configuration-part-4-pull-server/](http://hyper-v.nu/archives/bgelens/2015/02/integrating-vm-role-with-desired-state-configuration-part-4-pull-server/){:target="_blank"}

Series Index:
1. [Introduction and Scenario](http://bgelens.nl/integrating-vm-role-with-desired-state-configuration-part-1-introduction-and-scenario/)
2. [PKI](http://bgelens.nl/integrating-vm-role-with-desired-state-configuration-part-2-pki/)
3. [Creating the VM Role (step 1 and 2)](http://bgelens.nl/integrating-vm-role-with-desired-state-configuration-part-3-creating-the-vm-role-step-1-and-2/)
4. [Pull Server](http://bgelens.nl/integrating-vm-role-with-desired-state-configuration-part-4-pull-server/)
5. [Resource Kit](http://bgelens.nl/integrating-vm-role-with-desired-state-configuration-part-5-resource-kit/)
6. [PFX Repository Website](http://bgelens.nl/integrating-vm-role-with-desired-state-configuration-part-6-pfx-repository-website/)
7. [Creating a configuration document with encrypted content](http://bgelens.nl/integrating-vm-role-with-desired-state-configuration-part-7-creating-a-configuration-document-with-encrypted-content/)
8. [Create, deploy and validate the final VM Role](http://bgelens.nl/integrating-vm-role-with-desired-state-configuration-part-8-create,-deploy-and-validate-the-final-vm-role/)
9. [Create a Domain Join DSC resource](http://bgelens.nl/integrating-vm-role-with-desired-state-configuration-part-9-create-a-domain-join-dsc-resource/)
10. [Closing notes](http://bgelens.nl/integrating-vm-role-with-desired-state-configuration-part-10-closing-notes/)

In this post the DSC Pull Server will be created and configured with Client Certificate Authentication. Let’s look at the design first.

# DSC Pull Server Design

Again as with the PKI solution, we are dealing with a chicken and the egg situation. The company policy (explained in the introduction post) dictates no ad-hoc DSC configurations are allowed. All DSC configurations are only allowed to be deployed via a DSC Pull Server.  The DSC Pull Server will only be allowed to pass configuration documents (MOF files) if the LCM requesting such a file is trusted / authenticated. Also the DSC Pull Server itself should be trusted by the LCM instances for client certificate authentication to be available.

The DSC Pull Server website will be configured with an HTTPS binding only and it will be made available on the default HTTPS port (443) so it will be easy to make it available on-premises as well as on the internet. Because multiple websites will eventually be hosted on this server, Server Name Indication (SNI) will be enabled (host headers for HTTPS). The Web site will be configured to require both SSL and client authentication certificates.

The Web application pool associated with the DSC Pull Server website requires anonymous authentication to be available. When this is disabled, the website will actually not function so anonymous authentication will be configured.

![](/images/2015-02/BG_VMRole_DSC_P04P001.png)

Because the setting ‘require client authentication certificates’ on its own accepts client certificates provided by any of the trusted Certificate Authorities known by the webserver, the IIS Client Certificate Mapping component of IIS will be installed as well to restrict it a bit more. A client certificate mapping will be configured for the DSC Pull Server website to map many certificates to one account. The certificates allowed can only be explicitly provided by the Enterprise CA and because of this, an issuer rule will be configured making sure this is the case. An additional deny mapping will be configured to deny all other implicitly ‘trusted’ client certificates (certificates chained to any of the servers trusted Certificate Authority’s).

# Prerequisites

The Computer on which the DSC Pull Server gets deployed can either be domain joined or be a workgroup member. In my case I use another domain joined machine for simplicity reasons.

Since the Computer is joined to the same domain as the Enterprise CA, the computer has already have the Enterprise CA certificate in the Computer’s Trusted Root certificate store. If you choose not to deploy the DSC Pull Server on a domain joined computer, you need to import this certificate manually.

A Domain DNS zone should be created which reflects the external FQDN of the DSC Pull Web Service (as with the PKI solution).
* dscpull.domain.tld (dscpull.hyperv.nu)

# DSC Pull Server Installation Script

Let’s look at the installation script:

```powershell
#region params
param
(
    [Parameter(Mandatory=$true)]
    [PScredential]$CertificateCredentials,
    
    [Parameter(Mandatory=$true)]
    [String]$WebenrollURL,
    
    [Parameter(Mandatory=$true)]
    [String]$DSCPullFQDN
)
#endregion params
 
#region checks
if (!([Security.Principal.WindowsPrincipal] [Security.Principal.WindowsIdentity]::GetCurrent()).IsInRole([Security.Principal.WindowsBuiltInRole] "Administrator"))
{
    Write-Verbose 'Script can only run elevated' -Verbose
    break
}
#endregion checks
 
#region request webserver certificate
try
{
    $DSCPullCert = Get-Certificate -Url $WebenrollURL `
                                   -Template webserver `
                                   -SubjectName "CN=$DSCPullFQDN" `
                                   -DnsName $DSCPullFQDN `
                                   -CertStoreLocation Cert:\LocalMachine\My `
                                   -Credential $CertificateCredentials `
                                   -ErrorAction Stop `
                                   -Verbose
}
catch
{
    Write-Verbose 'Certificate Request did not complete successfully' -Verbose
    break
}
#endregion request webserver certificate
 
#region install roles and features
Install-WindowsFeature -Name Dsc-Service,Web-Cert-Auth -IncludeManagementTools
#endregion install roles and features
 
#region prepare website directory
$DestinationPath = (New-Item -Path C:\inetpub\wwwroot\PSDSCPullServer -ItemType directory -Force).FullName
$BinPath = (New-Item -Path $DestinationPath -Name 'bin' -ItemType directory -Force).FullName
$SourcePath = "$pshome/modules/psdesiredstateconfiguration/pullserver"
Copy-Item -Path $SourcePath\Global.asax -Destination $DestinationPath\Global.asax -Force | Out-Null
Copy-Item -Path $SourcePath\PSDSCPullServer.mof -Destination $DestinationPath\PSDSCPullServer.mof -Force | Out-Null
Copy-Item -Path $SourcePath\PSDSCPullServer.svc -Destination $DestinationPath\PSDSCPullServer.svc -Force | Out-Null
Copy-Item -Path $SourcePath\PSDSCPullServer.xml -Destination $DestinationPath\PSDSCPullServer.xml -Force | Out-Null
Copy-Item -Path $SourcePath\PSDSCPullServer.config -Destination $DestinationPath\web.config -Force | Out-Null
Copy-Item -Path $SourcePath\Microsoft.Powershell.DesiredStateConfiguration.Service.dll -Destination $BinPath -Force | Out-Null
#endregion prepare website directory
 
#region import webadmin ps module
Import-Module 'C:\Windows\system32\WindowsPowerShell\v1.0\Modules\WebAdministration\WebAdministration.psd1'
#endregion import webadmin ps module
 
#region configure IIS Aplication Pool
$AppPool = New-WebAppPool -Name PSWS -Force
$AppPool.processModel.identityType = 0 #configure app pool to run as local system
$AppPool.enable32BitAppOnWin64 = $true
$AppPool | Set-Item
#endregion configure IIS Aplication Pool
 
#region create site
$WebSite = New-Website -Name PSDSCPullServer `
                       -PhysicalPath $DestinationPath `
                       -ApplicationPool $AppPool.name `
                       -Port 443 `
                       -IPAddress * `
                       -Ssl `
                       -SslFlags 1 `
                       -HostHeader $DSCPullFQDN `
                       -Force
New-Item -Path "IIS:\SslBindings\!443!$DSCPullFQDN" -Value $DSCPullCert.Certificate -SSLFlags 1 | Out-Null
#endregion create site
 
#region unlock config data
Set-WebConfiguration -PSPath IIS:\ -Filter //access -Metadata overrideMode -value Allow -Force
Set-WebConfiguration -PSPath IIS:\ -Filter //anonymousAuthentication -Metadata overrideMode -value Allow -Force
Set-WebConfiguration -PSPath IIS:\ -Filter //basicAuthentication -Metadata overrideMode -value Allow -Force
Set-WebConfiguration -PSPath IIS:\ -Filter //windowsAuthentication -Metadata overrideMode -value Allow -Force
Set-WebConfiguration -PSPath IIS:\ -Filter //iisClientCertificateMappingAuthentication -Metadata overrideMode -value Allow -Force
#endregion unlock config data
 
#region setup application settings
Copy-Item -Path $pshome\Modules\PSDesiredStateConfiguration\PullServer\Devices.mdb -Destination $env:programfiles\WindowsPowerShell\DscService -Force
$configpath = "$env:programfiles\WindowsPowerShell\DscService\Configuration"
$modulepath = "$env:programfiles\WindowsPowerShell\DscService\Modules"
$jet4provider = "System.Data.OleDb"
$jet4database = "Provider=Microsoft.Jet.OLEDB.4.0;Data Source=$env:PROGRAMFILES\WindowsPowerShell\DscService\Devices.mdb;"
Add-WebConfigurationProperty -PSPath $WebSite.PSPath `
                             -Filter "appSettings" `
                             -Name '.' `
                             -Value @{key='dbprovider';value=$jet4provider} `
                             -Force
 
Add-WebConfigurationProperty -PSPath $WebSite.PSPath `
                             -Filter "appSettings" `
                             -Name '.' `
                             -Value @{key='dbconnectionstr';value=$jet4database} `
                             -Force
 
Add-WebConfigurationProperty -PSPath $WebSite.PSPath `
                             -Filter "appSettings" `
                             -Name '.' `
                             -Value @{key='ConfigurationPath';value= $configpath} `
                             -Force
                              
Add-WebConfigurationProperty -PSPath $WebSite.PSPath `
                             -Filter "appSettings" `
                             -Name '.' `
                             -Value @{key='ModulePath';value= $modulepath} `
                             -Force
#endregion setup application settings
 
#region require client auth certificates
Set-WebConfiguration -PSPath $WebSite.PSPath -Filter 'system.webserver/security/access' -Value "Ssl, SslNegotiateCert, SslRequireCert, Ssl128" -Force
#endregion require client auth certificates
 
#region create local user for Cert mapping
# nice simple password generation one-liner by G.A.F.F Jakobs
# https://gallery.technet.microsoft.com/scriptcenter/Simple-random-code-b2c9c9c9
$DSCUserPWD = ([char[]](Get-Random -Input $(48..57 + 65..90 + 97..122) -Count 12)) -join "" 
 
$Computer = [ADSI]"WinNT://.,Computer"
$DSCUser = $Computer.Create("User", "DSCUser")
$DSCUser.SetPassword($DSCUserPWD)
$DSCUser.SetInfo()
$DSCUser.Description = "DSC User for Client Certificate Authentication binding "
$DSCUser.SetInfo()
$DSCUser.UserFlags = 64 + 65536 # ADS_UF_PASSWD_CANT_CHANGE + ADS_UF_DONT_EXPIRE_PASSWD
$DSCUser.SetInfo()
([ADSI]"WinNT://./IIS_IUSRS,group").Add("WinNT://DSCUser,user")  
#endregion create local user for Cert mapping
 
#region configure certificate mapping
Add-WebConfigurationProperty -PSPath $WebSite.PSPath -Filter "system.webServer/security/authentication/iisClientCertificateMappingAuthentication/manyToOneMappings" -Name "." -Value @{name='DSC Pull Client';description='DSC Pull Client';userName='DSCUser';password=$DSCUserPWD}
Add-WebConfigurationProperty -PSPath $WebSite.PSPath -Filter "system.webServer/security/authentication/iisClientCertificateMappingAuthentication/manyToOneMappings/add[@name='DSC Pull Client']/rules" -Name "." -Value @{certificateField='Issuer';certificateSubField='CN';matchCriteria=$DSCPullCert.Certificate.Issuer.Split(',')[0].trimstart('CN=');compareCaseSensitive='False'}
Set-WebConfigurationProperty -PSPath $WebSite.PSPath -Filter "system.webServer/security/authentication/iisclientCertificateMappingAuthentication" -Name "enabled" -Value "True"
Set-WebConfigurationProperty -PSPath $WebSite.PSPath -Filter "system.webServer/security/authentication/iisclientCertificateMappingAuthentication" -Name "manyToOneCertificateMappingsEnabled" -Value "True"
Set-WebConfigurationProperty -PSPath $WebSite.PSPath -Filter "system.webServer/security/authentication/iisClientCertificateMappingAuthentication" -Name "oneToOneCertificateMappingsEnabled" -Value "False"
#endregion configure certificate mapping
 
#region configure deny other client certificates
Add-WebConfigurationProperty -PSPath $WebSite.PSPath -Filter "system.webServer/security/authentication/iisClientCertificateMappingAuthentication/manyToOneMappings" -name "." -value @{name='Deny';description='Deny';permissionMode='Deny'}
#endregion configure deny other client certificates

#region enable CAPI2 Operational Log
$logName = 'Microsoft-Windows-CAPI2/Operational'
$log = New-Object System.Diagnostics.Eventing.Reader.EventLogConfiguration $logName
$log.IsEnabled=$true
$log.SaveChanges()
#endregion enable CAPI2 Operational Log

#region remove default web site
Stop-Website -Name 'Default Web Site'
Remove-Website -Name 'Default Web Site'
#endregion remove default web site
```

To give credits where they belong, the script is partly based on a post blog post by [Steven Murawski](https://twitter.com/StevenMurawski){:target="_blank"} which you can find here: [http://powershell.org/wp/2013/10/03/building-a-desired-state-configuration-pull-server/](http://powershell.org/wp/2013/10/03/building-a-desired-state-configuration-pull-server/){:target="_blank"}.

So what does the script do?

* At the params region: it takes 3 parameters. CerificateCredentials (Domain User credentials to acquire a Web Server certificate from the Enterprise CA), WebEnrollURL (The WEB URL from where you can request Certificates from the Enterprise CA, I will use [https://webenroll.hyperv.nu/ADPolicyProvider_CEP_UsernamePassword/service.svc/CEP]()) and DSCPullFQDN (The FQDN on which the DSC Pull server will be made available, I will use dscpull.hyperv.nu).
* At the checks region: it will validate if the script is run in an elevated session (Run as Administrator). If not, script execution is aborted as elevation is mandatory to run the script.
* At the region request web server certificate: it will request a certificate based on the web server template using the webenroll URL (provided at the params region) and the DSC Pull Server FQDN (dscpull.hyperv.nu, provided at the params region) as the CN and SAN DNS. All Domain Authenticated Users should be able to request the certificate (see earlier PKI blog post). If the request does not succeed, script execution is aborted.

![](/images/2015-02/BG_VMRole_DSC_P04P002.png)

* At the region install roles and features: it will install all required roles and features including management tools.
* At the region prepare website directory: it will create the Pull Server web site directory under inetput wwwroot. In the Pull Server web directory, a bin directory is created. Required files are copied to the web directories.

![](/images/2015-02/BG_VMRole_DSC_P04P003.png)

![](/images/2015-02/BG_VMRole_DSC_P04P004.png)

* At the region configure IIS Application Pool: it creates an IIS application pool which runs as local system.

![](/images/2015-02/BG_VMRole_DSC_P04P005.png)

* At the region create site: it will create and configure the DSC Pull Server web site and binds the earlier requested certificate to it. The default HTTPS port (TCP 443) is used and the binding is configured for SNI as eventually multiple HTTPS sites will be hosted on this server.

![](/images/2015-02/BG_VMRole_DSC_P04P006.png)

* At the region unlock config data: it will unlock web.config sections needed by the DSC Pull Server website. Without unlocking, the settings later on cannot be made.
* At the region setup application settings: The DSC Pull Server application settings (database provider, connectionstring, configuration directory and module directory) are configured.

![](/images/2015-02/BG_VMRole_DSC_P04P007.png)

At the region require client auth certificates: it will configure the DSC Pull Server website to require SSL and Client certificates.

![](/images/2015-02/BG_VMRole_DSC_P04P008.png)

At the region create local user for cert mapping: it will create a local user which will be a member of the IIS_IUSRS group. This user will be used to configure certificate bindings rules to.

![](/images/2015-02/BG_VMRole_DSC_P04P009.png)

![](/images/2015-02/BG_VMRole_DSC_P04P010.png)

* At the region configure certificate mapping: it will enable client certificate authentication with many to one mapping.

![](/images/2015-02/BG_VMRole_DSC_P04P011.png)

It will add an Allow entry for the DSCUser (local user account) which was created earlier.

![](/images/2015-02/BG_VMRole_DSC_P04P012.png)

For the DSCUser it will add a rule for matching certificates issued by the Enterprise CA to the DSCUser user account.

![](/images/2015-02/BG_VMRole_DSC_P04P013.png)

* At the configure deny other client certificates region: it adds a second entry to map certificates to except in this case it won’t be configured to allow but to deny. No User account or rules will be configured as we only want to allow certificates from the Enterprise CA and want to deny everything else. This will work as a catchall for client authentication certificates not issued by the Enterprise CA but which are trusted via other trusted Certificate Authority’s.

![](/images/2015-02/BG_VMRole_DSC_P04P014.png)

* At the enable capi2 operational log region: it enables the CAPI2 event log which will be used to validate if the client authentication certificate is presented to the webserver and if it checks out correctly (disable after validation is done because this log can fill up rapidly).

![](/images/2015-02/BG_VMRole_DSC_P04P015.png)

* At the remove default website region: it stops and removes the default web site.

# Install the DSC Pull Web Service

So now you know what the script does, let’s run it on a workgroup (import the Enterprise CA certificate to the computer’s trusted root store first) or domain joined Windows Server 2012 R2 machine and validate if everything is configured correctly.

You can run the script though ISE or via the console.

![](/images/2015-02/BG_VMRole_DSC_P04P016.png)

When it is done, you can check the output stream for errors and validate the configuration using the screenshots provided in the script explanation section.

# Validate Secure DSC Pull Server functionality

To verify the DSC Pull Server functionality we will generate a simple configuration document which uses build in DSC resources only.

The configuration will be pulled by the earlier deployed VM Role VM on which we will (for this time) manually configure the LCM.

First the configuration:

```powershell
$GUID = [system.guid]::NewGuid().tostring()
configuration telnet
{
    node $GUID
    {
        WindowsFeature telnetclient
        {
            Ensure = 'Present'
            Name = 'Telnet-Client'
        }
    }
 
}
telnet -OutputPath 'C:\Program Files\WindowsPowerShell\DscService\Configuration' | Out-Null
New-DscCheckSum -Path 'C:\Program Files\WindowsPowerShell\DscService\Configuration' -Force
Write-Verbose -Message "For the LCM configuration use the following GUID:" -Verbose
Write-Verbose -Message $GUID -Verbose
```

This little script will generate a new GUID and then creates a configuration document for it. It declares the Telnet Client to be present.

A checksum file is created as well (the checksum file is used by the LCM to check if the configuration document has changed since the last time it had pulled it. If you apply a change in the configuration document, you have to update the checksum file so the LCM will pull the new config.

![](/images/2015-02/BG_VMRole_DSC_P04P018.png)

The GUID used is then reported back to you. You will need the GUID in the next step.

![](/images/2015-02/BG_VMRole_DSC_P04P019.png)

Note that since I also have WMF5 Preview installed on my DSC Pull Server on which I generated the configuration document, it is only compatible with WMF5 LCM instances because the MOF file schema has been updated with keys unknown by WMF4 LCM instances. See [http://powershell.org/wp/forums/topic/beware-windows-10-technical-preview/](http://powershell.org/wp/forums/topic/beware-windows-10-technical-preview/){:target="_blank"} for more info.

Next login to the VM Role VM deployed during the previous post and run the following configuration script.

```powershell
Configuration lcm
{
param
(
    [Parameter(Mandatory=$true)]
    [String]$GUID,
 
    [Parameter(Mandatory=$true)]
    [String]$PullServerURL
)
    node 'localhost'
    {
        LocalConfigurationManager
        {
            ConfigurationModeFrequencyMins = 15
            RefreshFrequencyMins = 30
            RebootNodeIfNeeded = $true
            ConfigurationMode =  'ApplyAndAutoCorrect'
            ConfigurationID = $GUID
            DownloadManagerCustomData = @{ServerUrl = "$PullServerURL/PSDSCPullServer.svc";
				                         AllowUnsecureConnection = 'true';
                                         CertificateID = (Get-ChildItem -Path Cert:\LocalMachine\My | ?{$_.FriendlyName -eq 'DSCPullServerAuthentication'}).Thumbprint;
                                         }
            DownloadManagerName = 'WebDownloadManager'
            RefreshMode = 'Pull'
        }
    }
}
Set-Location -Path C:\
lcm
Set-DscLocalConfigurationManager .\lcm -Verbose
Update-DscConfiguration -Wait -Verbose
```

Note that I’m using the v1 LCM configuration method as I don’t seem to get client certificate authentication to work with v2 (the LCM does not present the certificate to the Pull Server when using v2, I think it is a bug and I have reported it at connect here: [https://connect.microsoft.com/PowerShell/feedback/details/1112879/dsc-lcm-2-0-configuration-does-not-work-with-client-certificate-authentication](https://connect.microsoft.com/PowerShell/feedback/details/1112879/dsc-lcm-2-0-configuration-does-not-work-with-client-certificate-authentication){:target="_blank"}).

It will take the GUID generated at the previous step (in my case: 2d958c0d-6168-4678-bc9f-df1ec2ce476c) and the DSC Pull Server URL (in my case: [https://dscpull.hyperv.nu]()). While generating the LCM configuration document it looks for the client certificate by using a predefined friendlyname. By now you will probably know how we will configure the rest of the VM Role right? The LCM instance is then configured and an update is forced. When everything works correctly you will see a lot of output about the LCM applying the configuration and installing the telnet client.

![](/images/2015-02/BG_VMRole_DSC_P04P020.png)

So it is now validated the LCM is getting its configuration document from the pull server using a client authentication certificate. Let’s look at the Security and CAP2 log on the pull server. First the CAP2 log:

![](/images/2015-02/BG_VMRole_DSC_P04P021.png)

If the LCM is presenting a client authentication certificate you will see a couple of events. A few about the revocation check of the certificate, one about the certificate itself and a few which validate if the certificate chain is trusted. When opening the event with ID 90 we can see which certificate was presented and all info about its intended usage and issuer.

![](/images/2015-02/BG_VMRole_DSC_P04P022.png)

We then look at the Security log:

![](/images/2015-02/BG_VMRole_DSC_P04P023.png)

We can see the certificate logon resulted in the logon of the DSCUser for which the certificate mapping was created.

In the W3SVC Log located at C:\inetpub\logs\LogFiles\W3SVCx we can see a 200 event which stands for success.

I actually added an extra Enterprise CA to validate if the certificate mapping works as intended (when a certificate is trusted by the webserver but does not hit the allow rule, it should be denied).

On the LCM instance I received a forbidden:

![](/images/2015-02/BG_VMRole_DSC_P04P024.png)

On the CAPI2 log I can see the certificate being presented to the web server and validated correctly but at the W3SVC log I see the request getting a 403.12 status which means Mapper denied access. Exactly what we are aiming for. For more info about IIS status codes see: [http://support2.microsoft.com/kb/943891/en](http://support2.microsoft.com/kb/943891/en){:target="_blank"}.

When you disable the certificate mapping you the LCM will have access. The certificate will then be used to login the IUSR account (anonymous).

That’s it for this post. Next we will populate the DSC Pull Server modules with all modules available in the latest resource kit (v9 at the time of writing).