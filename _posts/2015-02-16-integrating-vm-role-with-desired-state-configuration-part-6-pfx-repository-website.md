---
title:  "Integrating VM Role with Desired State Configuration Part 6 – PFX Repository Website"
date:   2015-02-16 12:00:00
categories: ["WAPACK","Windows Azure Pack","Desired State Configuration","PSDSC"]
tags: ["WAPACK","Windows Azure Pack","PowerShell","VM Role","PKI","PSDSC","Desired State Configuration"]
---

This blog post is written and published on Hyper-V.nu: [http://hyper-v.nu/archives/bgelens/2015/02/integrating-vm-role-with-desired-state-configuration-part-6-pfx-repository-website/](http://hyper-v.nu/archives/bgelens/2015/02/integrating-vm-role-with-desired-state-configuration-part-6-pfx-repository-website/){:target="_blank"}

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

In this post the PFX Repository website is created which is accessed during VM deployment to download a PFX container belonging to the configuration ID. As a DSC configuration ID can be assigned to many LCM instances simultaneously, the client authentication certificate cannot be used for configuration document encryption purposes as these certificates are unique for each instance.

# PFX Website and functionality design

Every configuration document published to the DSC Pull Server will have an associated PFX container containing both the public and private key pairs used to encrypt / decrypt any potential sensitive data included in the document. If the configuration document currently does not have sensitive data, a PFX is issued nonetheless as sensitive data could be added in a later stage.

The PFX Website will be available over HTTPS only and will require client certificate authentication to be accessed. The client authentication certificates assigned to the VMs during deployment will be the allowed certificates.

A unique pin code used for creating and opening a PFX file will be available via the website as well. In my opinion this is a better practice then using a pre-defined pin code for all PFX files. It is still not the ultimate solution but I think the route taken is secure enough for now. If you have suggestions improving this bit please reach out!

The certificate containing the public key will be saved to a repository available to the configuration document generators. For now this will be a local directory.

# Prerequisites

The Computer on which the PFX Website gets deployed can either be domain joined or be a workgroup member. In my case I use the DSC Pull Server from the previous post as I don’t have a lot of resources.

Since the Computer is joined to the same domain as the Enterprise CA, the computer has already have the Enterprise CA certificate in the Computer’s Trusted Root certificate store. If you choose not to deploy the PFX Website on a domain joined computer, you need to import this certificate manually.

A Domain DNS zone should be created which reflects the external FQDN of the PFX Website.
* pfx.domain.tld (pfx.hyperv.nu)

# PFX Website Installation Script

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
    [String]$PFXFQDN
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
    $PFXWebCert = Get-Certificate -Url $WebenrollURL `
                                  -Template webserver `
                                  -SubjectName "CN=$PFXFQDN" `
                                  -DnsName $PFXFQDN `
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
Install-WindowsFeature -Name Web-Server,Web-Cert-Auth -IncludeManagementTools
#endregion install roles and features
 
#region prepare website directory
$DestinationPath = (New-Item -Path C:\PFXSite -ItemType directory -Force).FullName
#endregion prepare website directory
 
#region import webadmin ps module
Import-Module 'C:\Windows\system32\WindowsPowerShell\v1.0\Modules\WebAdministration\WebAdministration.psd1'
#endregion import webadmin ps module
 
#region configure IIS Aplication Pool
$AppPool = New-WebAppPool -Name PFXWS -Force
#endregion configure IIS Aplication Pool
 
#region create site
$WebSite = New-Website -Name PFX `
                       -PhysicalPath $DestinationPath `
                       -ApplicationPool $AppPool.name `
                       -Port 443 `
                       -IPAddress * `
                       -Ssl `
                       -SslFlags 1 `
                       -HostHeader $PFXFQDN `
                       -Force
New-Item -Path "IIS:\SslBindings\!443!$PFXFQDN" -Value $PFXWebCert.Certificate -SSLFlags 1 | Out-Null
#endregion create site
 
#region unlock config data
Set-WebConfiguration -PSPath IIS:\ -Filter //access -Metadata overrideMode -value Allow -Force
Set-WebConfiguration -PSPath IIS:\ -Filter //iisClientCertificateMappingAuthentication -Metadata overrideMode -value Allow -Force
#endregion unlock config data

#region disabe anonymous logon
Set-WebConfigurationProperty -PSPath $WebSite.PSPath  -Filter "system.webServer/security/authentication/anonymousAuthentication" -Name "enabled" -Value "False" -Force
#endregion disable anonymous logon
  
#region require client auth certificates
Set-WebConfiguration -PSPath $WebSite.PSPath -Filter 'system.webserver/security/access' -Value "Ssl, SslNegotiateCert, SslRequireCert, Ssl128" -Force
#endregion require client auth certificates
 
#region create local user for Cert mapping
# nice simple password generation one-liner by G.A.F.F Jakobs
# https://gallery.technet.microsoft.com/scriptcenter/Simple-random-code-b2c9c9c9
$PFXUserPWD = ([char[]](Get-Random -Input $(48..57 + 65..90 + 97..122) -Count 12)) -join "" 
 
$Computer = [ADSI]"WinNT://.,Computer"
$PFXUser = $Computer.Create("User", "PFXUser")
$PFXUser.SetPassword($PFXUserPWD)
$PFXUser.SetInfo()
$PFXUser.Description = "PFX User for Client Certificate Authentication binding "
$PFXUser.SetInfo()
$PFXUser.UserFlags = 64 + 65536 # ADS_UF_PASSWD_CANT_CHANGE + ADS_UF_DONT_EXPIRE_PASSWD
$PFXUser.SetInfo()
([ADSI]"WinNT://./IIS_IUSRS,group").Add("WinNT://PFXUser,user")  
#endregion create local user for Cert mapping
 
#region configure certificate mapping
Add-WebConfigurationProperty -PSPath $WebSite.PSPath -Filter "system.webServer/security/authentication/iisClientCertificateMappingAuthentication/manyToOneMappings" -Name "." -Value @{name='PFX Web Client';description='PFX Web Client';userName='PFXUser';password=$PFXUserPWD}
Add-WebConfigurationProperty -PSPath $WebSite.PSPath -Filter "system.webServer/security/authentication/iisClientCertificateMappingAuthentication/manyToOneMappings/add[@name='PFX Web Client']/rules" -Name "." -Value @{certificateField='Issuer';certificateSubField='CN';matchCriteria=$PFXWebCert.Certificate.Issuer.Split(',')[0].trimstart('CN=');compareCaseSensitive='False'}
Set-WebConfigurationProperty -PSPath $WebSite.PSPath -Filter "system.webServer/security/authentication/iisclientCertificateMappingAuthentication" -Name "enabled" -Value "True"
Set-WebConfigurationProperty -PSPath $WebSite.PSPath -Filter "system.webServer/security/authentication/iisclientCertificateMappingAuthentication" -Name "manyToOneCertificateMappingsEnabled" -Value "True"
Set-WebConfigurationProperty -PSPath $WebSite.PSPath -Filter "system.webServer/security/authentication/iisClientCertificateMappingAuthentication" -Name "oneToOneCertificateMappingsEnabled" -Value "False"
#endregion configure certificate mapping
 
#region configure deny other client certificates
Add-WebConfigurationProperty -PSPath $WebSite.PSPath -Filter "system.webServer/security/authentication/iisClientCertificateMappingAuthentication/manyToOneMappings" -name "." -value @{name='Deny';description='Deny';permissionMode='Deny'}
#endregion configure deny other client certificates
 
#region set WebFolder ACL
$Acl = Get-Acl -Path C:\PFXSite
$Ar = New-Object System.Security.AccessControl.FileSystemAccessRule("PFXUser","ReadAndExecute, Synchronize","ContainerInherit, ObjectInherit","None","Allow")
$Acl.SetAccessRule($Ar)
Set-Acl -Path C:\PFXSite -AclObject $Acl
#endregion set WebFolder ACL
```

So what does the script do?

* At the params region: it takes 3 parameters. CerificateCredentials (Domain User credentials to acquire a Web Server certificate from the Enterprise CA), WebEnrollURL (The WEB URL from where you can request Certificates from the Enterprise CA, I will use [https://webenroll.hyperv.nu/ADPolicyProvider_CEP_UsernamePassword/service.svc/CEP]()) and PFXFQDN (The FQDN on which the PFX Web server will be made available, I will use pfx.hyperv.nu).
* At the checks region: it will validate if the script is run in an elevated session (Run as Administrator). If not, script execution is aborted as elevation is mandatory to run the script.
* At the region request web server certificate: it will request a certificate based on the web server template using the webenroll URL (provided at the params region) and the PFX Web Server FQDN (pfx.hyperv.nu, provided at the params region) as the CN and SAN DNS. All Domain Authenticated Users should be able to request the certificate (see earlier PKI blog post). If the request does not succeed, script execution is aborted.

![](/images/2015-02/BG_VMRole_DSC_P06P001.png)

* At the region install roles and features: it will install all required roles and features including management tools.
* At the region prepare website directory: it will create the PFX web site directory under the root of C:\.
* At the region configure IIS Application Pool: it creates an IIS application pool.

![](/images/2015-02/BG_VMRole_DSC_P06P002.png)

* At the region create site: it will create and configure the PFX web site and binds the earlier requested certificate to it. The default HTTPS port (TCP 443) is used and the binding is configured for SNI as multiple HTTPS sites are hosted on this server (DSC Pull Server).

![](/images/2015-02/BG_VMRole_DSC_P06P003.png)

* At the region unlock config data: it will unlock web.config sections needed by the PFX website. Without unlocking, the settings later on cannot be made.
* At the region disable anonymous access: it will disable anonymous access.

![](/images/2015-02/BG_VMRole_DSC_P06P004.png)

* At the region require client auth certificates: it will configure the PFX website to require SSL and Client certificates.

![](/images/2015-02/BG_VMRole_DSC_P06P005.png)

* At the region create local user for cert mapping: it will create a local user which will be a member of the IIS_IUSRS group. This user will be used to configure certificate bindings rules to and to get data from the website directory.

![](/images/2015-02/BG_VMRole_DSC_P06P006.png)

![](/images/2015-02/BG_VMRole_DSC_P06P007.png)

* At the region configure certificate mapping: it will enable client certificate authentication with many to one mapping.

![](/images/2015-02/BG_VMRole_DSC_P06P008.png)

It will add an Allow entry for the PFXUser (local user account) which was created earlier.

![](/images/2015-02/BG_VMRole_DSC_P06P009.png)

It will add a rule for matching certificates issued by the Enterprise CA to the PFXUser user account.

![](/images/2015-02/BG_VMRole_DSC_P06P010.png)

* At the configure deny other client certificates region: it adds a second entry to map certificates to except in this case it won’t be configured to allow but to deny. No User account or rules will be configured as we only want to allow certificates from the Enterprise CA and want to deny everything else. This will work as a catchall for client authentication certificates not issued by the Enterprise CA but which are trusted via other trusted Certificate Authority’s.

![](/images/2015-02/BG_VMRole_DSC_P06P011.png)

At the set webfolder acl region: it will add the needed permission for the PFXUser to the PFX web site folder.

![](/images/2015-02/BG_VMRole_DSC_P06P012.png)

# Install the PFX Website

You can run the script though ISE or via the console.

![](/images/2015-02/BG_VMRole_DSC_P06P013.png)

When it is done, you can check the output stream for errors and validate the configuration using the screenshots provided in the script explanation section.

# Validate PFX Website functionality

To validate the functionality of the PFX website we will add a text file to the pfx web site folder called test.txt with some random tekst.

Next we run the following command on the VM Role VM deployed earlier.

```powershell
Invoke-WebRequest https://pfx.hyperv.nu/test.txt -OutFile c:\test.txt -Verbose -Certificate (Get-ChildItem -Path Cert:\LocalMachine\My\ | ?{$_.FriendlyName -eq 'DSCPullServerAuthentication'})
```

![](/images/2015-02/BG_VMRole_DSC_P06P014.png)

When the test.txt file is downloaded, functionality is tested successfully.

As with the DSC Pull Server verification, you can verify the CAPI2 and security logs.

![](/images/2015-02/BG_VMRole_DSC_P06P015.png)

That’s it for this post. Next we will create a PFX file and generate a configuration document with encrypted sensitive data.