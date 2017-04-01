---
title:  "Integrating VM Role with Desired State Configuration Part 2 – PKI"
date:   2015-02-06 12:00:00
categories: ["WAPACK","Windows Azure Pack","Desired State Configuration","PSDSC"]
tags: ["WAPACK","Windows Azure Pack","PowerShell","VM Role","PKI","PSDSC","Desired State Configuration"]
---

This blog post is written and published on Hyper-V.nu: [http://hyper-v.nu/archives/bgelens/2015/02/integrating-vm-role-with-desired-state-configuration-part-2-pki/](http://hyper-v.nu/archives/bgelens/2015/02/integrating-vm-role-with-desired-state-configuration-part-2-pki/){:target="_blank"}

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

In this post, a brand new PKI tailored to fit the needs of this blog series will be implemented. Let’s take a look at the “design” first.

# PKI Design

The PKI solution described in this post will be implemented solely to support the VM Role DSC implementation as described in part 1. The design focuses on functionality first. PKI best practices and security guidelines are not part of the design goals (at least not explicitly).

The PKI will exists out of the following components:

* Enterprise Root Certificate Authority
* CDP / AIA Website
* Web Enrollment Policy Web Service (CEP)
* Web Enrollment Web Service (CES)

To make sure the impact on a potentially already existing Enterprise Root CA will be minimized, a new Enterprise Root CA will be deployed. This will make sure the CA is implemented in such a way that it is workable for the scenario where the DSC Pull Client is not (or not yet) a member of the Active Directory Domain. A Standalone CA cannot be used as Web Enrollment Services will be implemented which requires an Enterprise CA.

Because the DSC Pull Client does not have to be a member of the domain, the CDP and AIA locations added to issued certificates will be targeted to web locations only. This will prevent unnecessary lookups against the Active Directory CDP and AIA containers to which the client does not have access at the time of deployment or will not ever have access in the case it won’t be joined to the domain. Plus this setup will make sure the CA’s CRL and AIA locations are accessible publicly as well. For more info about CDP and AIA see: [http://technet.microsoft.com/en-us/library/cc776904(v=ws.10).aspx](http://technet.microsoft.com/en-us/library/cc776904(v=ws.10).aspx){:target="_blank"}.

The websites for CDP / AIA and Web Enrollment Services will be resolvable using public DNS information. For this scenario external DNS is simulated by adding a Public DNS name zone to the internal DNS. The web address for the Pull Server will eventually be resolvable via this method as well as it too should be available internally and externally.

>All endpoints required for the VM Role with DSC Pull Server integration scenario described in this series are web based. Because of this, the endpoints are easily made accessible when utilizing network virtualization or when deploying DSC Pull clients to a public cloud service (e.g. Azure or ISP) using other / similar mechanisms as the VM Role. When comparing to the Azure DSC VM Extension, it depends on internet connectivity as it need to get it’s configuration document and payload from Azure blob storage. Blob storage is external from the VM which is why the Extension will do a web call to acquire it. The public accessibility of services (which I think will get more common as we venture further into distributed deployments over private, hybrid and public cloud environments) make it ever more crucial for proper encryption and authentication mechanisms to be in place.

Web Enrollment Services will be used to provide access to Enterprise CA functionality (Enrollment based on templates) without the need for the requestor (client) to directly talk to the CA / Active Directory or be a member of the Active Directory. Instead, access to the Enrollment Services is handled over HTTPS and the Enrollment Services act as a protocol proxy on behalf of the requestor. The Web Enrollment Services can be implemented in a variety of authentication configurations. Again, because of the clients requesting the certificate do not per se belong to the Active Directory domain, authentication will be configured to require a username password pair instead of Kerberos or client certificates. The Web Enrollment Services will make use of a certificate with server authentication EKU to provide for an HTTPS binding. The certificate will be issued by the Enterprise Root CA deployed in this scenario. For more information about Certificate Enrollment Web Services see: [http://technet.microsoft.com/en-us/library/hh831822.aspx](http://technet.microsoft.com/en-us/library/hh831822.aspx){:target="_blank"}.

# Prerequisites

To start implementing the PKI there should already be an Active Directory domain available as well as a domain joined computer installed with Windows Server 2012 R2, which will be the platform for the CA installation. Two DNS name zones should be created which reflects the external FQDN’s (for this example I’ll be using hyper.v.nu domain).

* cdp.domain.tld (cdp.hyperv.nu)
* webenroll.domain.tld (webenroll.hyperv.nu)

Both zones will have an A record with an empty hostname pointing to the IP address of the domain joined computer on which the CA will be installed. The names can be anything you like. I create two zones so it will not conflict with the Public DNS information. For example: as the zone is authoritative only for the cdp.domain.tld, information about domain.tld will still be queried from the internet.

![](/images/2015-02/BG_VMRole_DSC_P02P001.png)

You could off course use a Public DNS instead so you actually implement an end to end solution without the need for emulation.

# PKI Implementation script

Since we are dealing with a chicken and egg situation, as the DSC Pull server is not in place yet, the CA will be deployed using a simple PowerShell script and not via a DSC configuration document. Let’s look at the script first.

```powershell
#region params
Param 
(
    [Parameter(Mandatory=$true)]
    [String]$CAName,
 
    [Parameter(Mandatory=$true)]
    [String]$CDPURL,
 
    [Parameter(Mandatory=$true)]
    [String]$WebenrollURL
)
#endregion params

#region normalize URL to FQDN
if ($CDPURL -like "http://*" -or $CDPURL -like "https://*")
{
    $CDPURL = $CDPURL.Split('/')[2]
 
}
 
if ($WebenrollURL -like "http://*" -or $WebenrollURL -like "https://*")
{
    $WebenrollURL = $WebenrollURL.Split('/')[2]
 
}
#endregion normalize URL to FQDN 
 
#region checks
if (!([Security.Principal.WindowsPrincipal] [Security.Principal.WindowsIdentity]::GetCurrent()).IsInRole([Security.Principal.WindowsBuiltInRole] "Administrator"))
{
    Write-Verbose 'Script can only run elevated' -Verbose
    break
}
$CurrentUser = [System.Security.Principal.WindowsIdentity]::GetCurrent() 
$WindowsPrincipal = New-Object System.Security.Principal.WindowsPrincipal($CurrentUser)
if (!($WindowsPrincipal.IsInRole('Enterprise Admins')))
{
    Write-Verbose 'Script can only run with Enterprise Administrator privileges' -Verbose
    break
}
#endregion checks
 
#region install required roles and features
Install-WindowsFeature -Name ADCS-Cert-Authority,ADCS-Enroll-Web-Pol,ADCS-Enroll-Web-Svc -IncludeManagementTools
#endregion install required roles and features
 
#region Install Enterprise Root CA
try
{
    Install-AdcsCertificationAuthority -WhatIf
}
catch
{
    Write-Verbose 'A CA is already installed on this server, cleanup server and AD before running this script again' -Verbose
    break
}
if ((certutil -adca |Select-String "cn =").line.Substring(7) -contains $CAName)
{
    Write-Verbose "An Enterprise CA with the CA Name $CAName already exists" -Verbose
    break
}
 
 
New-Item C:\Windows\capolicy.inf -ItemType file -Force | Out-Null
@"
[Version]
Signature="`$Windows NT$"
 
[PolicyStatementExtension]
Policies=InternalUseOnly
[InternalUseOnly]
OID=2.5.29.32.0
Notice="This CA is used for a DSC demo environment"
 
[Certsrv_Server]
LoadDefaultTemplates=0
AlternateSignatureAlgorithm=1
 
[Extensions]
2.5.29.15 = AwIBBg==
Critical = 2.5.29.15
"@ | Out-File C:\Windows\capolicy.inf -Force
 
Install-AdcsCertificationAuthority -CACommonName $CAName `
                                   -CAType EnterpriseRootCA `
                                   -CADistinguishedNameSuffix 'O=DSCCompany,C=NL' `
                                   -HashAlgorithmName sha256 `
                                   -ValidityPeriod Years `
                                   -ValidityPeriodUnits 10 `
                                   -CryptoProviderName 'RSA#Microsoft Software Key Storage Provider' `
                                   -KeyLength 4096 `
                                   -Force

certutil -setreg CA\AuditFilter 127
certutil -setreg CA\ValidityPeriodUnits 4
certutil -setreg CA\ValidityPeriod "Years"
#endregion Install Enterprise Root CA
 
#region configure CA settings and prepare AIA / CDP
New-Item c:\CDP -ItemType directory -Force
Copy-Item C:\Windows\System32\CertSrv\CertEnroll\*.crt C:\CDP\$CAName.crt -Force
Get-CAAuthorityInformationAccess | Remove-CAAuthorityInformationAccess -Force
Get-CACrlDistributionPoint | Remove-CACrlDistributionPoint -Force
Add-CAAuthorityInformationAccess -Uri http://$CDPURL/$CAName.crt -AddToCertificateAia -Force
Add-CACrlDistributionPoint -Uri C:\CDP\<CAName><CRLNameSuffix><DeltaCRLAllowed>.crl -PublishToServer -PublishDeltaToServer -Force
Add-CACrlDistributionPoint -Uri http://$CDPURL/<CAName><CRLNameSuffix><DeltaCRLAllowed>.crl -AddToCertificateCdp -AddToFreshestCrl -Force
#endregion configure CA settings and prepare AIA / CDP
 
#region create CDP / AIA web site
Import-Module 'C:\Windows\system32\WindowsPowerShell\v1.0\Modules\WebAdministration\WebAdministration.psd1'
New-Website -Name CDP -HostHeader $CDPURL -Port 80 -IPAddress * -Force
Set-ItemProperty 'IIS:\Sites\CDP' -Name physicalpath -Value C:\CDP
Set-WebConfigurationProperty -PSPath 'IIS:\Sites\CDP' -Filter /system.webServer/directoryBrowse  -Name enabled -Value true
Set-WebConfigurationProperty -PSPath 'IIS:\Sites\CDP' -Filter /system.webServer/security/requestfiltering  -Name allowDoubleEscaping -Value true
attrib +h C:\CDP\web.config
#endregion create CDP / AIA web site
 
#region restart CA service and publish CRL
Restart-Service -Name CertSvc
Start-Sleep -Seconds 5
certutil -CRL
#endregion restart CA service and publish CRL
 
#region add webserver template
Invoke-Command -ComputerName ($env:LOGONSERVER).Trim("\") -ScriptBlock {
    $DN = (Get-ADDomain).DistinguishedName
    $WebTemplate = "CN=WebServer,CN=Certificate Templates,CN=Public Key Services,CN=Services,CN=Configuration,$DN"
    DSACLS $WebTemplate /G "Authenticated Users:CA;Enroll"
 
}
 
certutil -setcatemplates +WebServer
#endregion add webserver template
 
#region request web server certificate
$cert = Get-Certificate -Template webserver -DnsName $webenrollURL -SubjectName "CN=$webenrollURL" -CertStoreLocation cert:\LocalMachine\My
#endregion request web server certificate
 
#region Install enrollment web services
Install-AdcsEnrollmentPolicyWebService -AuthenticationType UserName -SSLCertThumbprint $cert.Certificate.Thumbprint -Force
Set-WebConfigurationProperty -pspath 'MACHINE/WEBROOT/APPHOST/Default Web Site/ADPolicyProvider_CEP_UsernamePassword'  -filter "appSettings/add[@key='FriendlyName']" -name "value" -value "DSC CA" -Force
Install-AdcsEnrollmentWebService -AuthenticationType UserName -SSLCertThumbprint $cert.Certificate.Thumbprint -Force
#endregion Install enrollment web services
 
#region modify Enrollment Server URL in AD
Invoke-Command -ComputerName ($env:LOGONSERVER).Trim("\") -ScriptBlock {
    param
    (
        $CAName,
        $webenrollURL
    )
    $DN = (Get-ADDomain).DistinguishedName
    $CAEnrollmentServiceDN = "CN=$CAName,CN=Enrollment Services,CN=Public Key Services,CN=Services,CN=Configuration,$DN"
    Set-ADObject $CAEnrollmentServiceDN -Replace @{'msPKI-Enrollment-Servers'="1`n4`n0`nhttps://$webenrollURL/$CAName`_CES_UsernamePassword/service.svc/CES`n0"}
} -ArgumentList $CAName, $webenrollURL
#endregion modify Enrollment Server URL in AD
```

So what does the script do?

* At the params region: it takes 3 parameters. The CA name (I will use DSCCA as the value), CDP URL (cdp.hyperv.nu) and WebEnroll URL (webenroll.hyperv.nu).
* At the checks region: it will validate if the script is run in an elevated session (Run as Administrator). If not, script execution is aborted as elevation is mandatory to run the script. Next it checks if the user running the script is a member of the Enterprise Admins group. Again a mandatory requirement as installation of an Enterprise CA requires this permission. If the user does not meet this requirement, script execution is aborted.
* At the region install required roles ….: it will install all required roles and features including management tools.
* At the region install Enterprise CA: it will check if a CA is already installed on the current machine. If a CA is already installed an exception is generated which terminates script execution. Then the script will check if an Enterprise CA with the provided name already exists. If it does, again the script is terminated. If all checks turn out false a capolicy.inf file will be created containing customization settings which are parsed during CA installation (for more info about capolicy.inf see: [http://blogs.technet.com/b/askds/archive/2009/10/15/windows-server-2008-r2-capolicy-inf-syntax.aspx](http://blogs.technet.com/b/askds/archive/2009/10/15/windows-server-2008-r2-capolicy-inf-syntax.aspx){:target="_blank"}). Finally the CA is installed using the provided CA name and with some static parameter values. When the CA is installed the auditfilter is set as well as the maximum possible lifetime of issued certificates.

![](/images/2015-02/BG_VMRole_DSC_P02P002.png)

* At the region configure CA settings and prepare AIA / CDP: it will create a directory for the CDP website content and copy’s the public key of the Enterprise Root CA to it while renaming it.

![](/images/2015-02/BG_VMRole_DSC_P02P003.png)

All default AIA and CDP extension information is removed so certificates which will be issued won’t have any reference to LDAP locations. The CDP folder will be configured at the CDP extension for publishing CRL and delta CRL’s.

![](/images/2015-02/BG_VMRole_DSC_P02P004.png)

The AIA and CDP extensions will be configured with the CDP URL provided by the parameters at the beginning of the script. The URLs will be added to the certificate data when they are issued.
* At the region Create CDP / AIA web site: it will create the CDP website with a binding on port 80 (note that CDP and AIA websites cannot be configured with HTTPS ).

![](/images/2015-02/BG_VMRole_DSC_P02P005.png)

The website will be configured with the physical path of the CDP directory created at the previous script region.

![](/images/2015-02/BG_VMRole_DSC_P02P006.png)

Directory browsing and double escaping will be enabled which are required for the purpose of the website.

![](/images/2015-02/BG_VMRole_DSC_P02P007.png)

![](/images/2015-02/BG_VMRole_DSC_P02P008.png)

The web.config will be configured with the hidden attribute so it will not appear if the site is accessed via directory browsing.
* At the region restart CA service and publish CRL: it restarts the CA service to activate all new settings and the first CRL is published to the CDP directory and thus will be available via the CDP URL.

![](/images/2015-02/BG_VMRole_DSC_P02P009.png)

* At the region add webserver template: it will remote to the domain controller which is known as the logon server and adds the enroll permission for Authenticated Users to the default web server template (For not making this blog series overcomplicated, we use authenticated users enroll permission for all templates we create. You could (and actually should) of course make a domain group with explicitly defined members for every template).

![](/images/2015-02/BG_VMRole_DSC_P02P010.png)

Then the webserver template is added to the CA’s certificate templates to issue.

![](/images/2015-02/BG_VMRole_DSC_P02P011.png)

* At the region request web server certificate: it will request a certificate based on the web server template providing the webenroll URL (provided at the params region) as the DNS name and Common Name. The certificate will be issued as Authenticated Users have enroll permission which were set at the previous section. The certificate will be stored in the computer certificate store as it needs to be available for IIS later on.

![](/images/2015-02/BG_VMRole_DSC_P02P012.png)

* At the region Install enrollment web services: it will install both the web enrollment service and web enrollment policy service. Both will have a HTTPS binding utilizing the certificate enrolled at the previous region.

![](/images/2015-02/BG_VMRole_DSC_P02P013.png)

The friendly name for the policy server will be configured (for more information about this see: [http://technet.microsoft.com/en-us/library/hh831625.aspx#feedback](http://technet.microsoft.com/en-us/library/hh831625.aspx#feedback){:target="_blank"}).

![](/images/2015-02/BG_VMRole_DSC_P02P014.png)

* At the region modify enrollment server URL in AD: it will remote to the domain controller which is known as the logon server and configures the correct URL for the web enrollment services (the client will connect with the policy server and it will eventually refer the client to the URL known at the attribute which is configured for enrollment).

![](/images/2015-02/BG_VMRole_DSC_P02P015.png)

# Install the CA and Web Enrollment Services

So now you know what the script does, let’s run it on a domain joined Windows Server 2012 R2 machine and validate if everything is configured correctly.

You can run the script though ISE or via the console.

![](/images/2015-02/BG_VMRole_DSC_P02P016.png)

When it is done, you can check the output stream for errors (I never have any). Next validate the PKI by running pkiview.msc on the server.

![](/images/2015-02/BG_VMRole_DSC_P02P017.png)

If any misconfigurations / errors exists concerning trust chain and CDP / AIA locations they will show up here.

Any other validations can be done using the screenshots available in the script explanation section.

# Create the DSC Client Certificate Template

Now the PKI is available, the DSC Client Certificate template can be created and made available. I won’t go into detail very much as it is stated at the start of this post that we will focus on functionality first.

First open the Certificate Templates Console by typing certtmpl.msc. Then select the “Workstation Authentication” template and create a duplicate.

![](/images/2015-02/BG_VMRole_DSC_P02P018.png)

Change the compatibility settings for Certificate Authority and Recipient to Windows Server 2012 R2.

![](/images/2015-02/BG_VMRole_DSC_P02P019.png)

On the General tab give the template a meaningful name. This name will be used later on by the client to request the certificate. Configure the validity period (I choose 4 years as it is configured as the maximum allowed validity period for the CA during the CA setup).

![](/images/2015-02/BG_VMRole_DSC_P02P020.png)

On the Request Handling tab change the purpose to Signature. This will better prevent the Certificate for being used for other purposes then its intended purpose (e.g. this certificate should only be used for authentication and not for encryption).

![](/images/2015-02/BG_VMRole_DSC_P02P021.png)

On the Crypthography tab:

* Change the Provider Category to Key Storage Provider
* Set the minimum key size to 4096
* Change the request hash to SHA256

![](/images/2015-02/BG_VMRole_DSC_P02P022.png)

On the Subject Name tab select Supply in the request.

![](/images/2015-02/BG_VMRole_DSC_P02P023.png)

On the Extensions Tab select Application Policies. At the description field you will see the most important setting of the template “Client Authentication” being configured as an extended key usage.

![](/images/2015-02/BG_VMRole_DSC_P02P024.png)

Next select Key Usage and select edit. Deselect Signature is proof of origin (nonrepudiation) and select OK.

![](/images/2015-02/BG_VMRole_DSC_P02P025.png)

On the Security tab select Authenticated Users and Allow the Enroll permission.

![](/images/2015-02/BG_VMRole_DSC_P02P026.png)

Now the template is finished. Select OK.

# Add Certificate Template to Templates to Issue

Open the Certificate Authority console by typing certsrv.msc.

Navigate to the Certificate Template node, right click it and select New -> Certificate Template to Issue.

![](/images/2015-02/BG_VMRole_DSC_P02P027.png)

Select the Certificate template which was created earlier and press OK.

![](/images/2015-02/BG_VMRole_DSC_P02P028.png)

Now 2 templates are available for the CA to issue.

![](/images/2015-02/BG_VMRole_DSC_P02P029.png)

# Validate Web Enrollment Services

To verify the functionality of the web enrollment services run the following command on the CA.

```powershell
$credentials = Get-Credential -Message 'Enter Valid Domain Credentials'
$webenrollURL = 'webenroll.hyperv.nu'
$templateName = 'DSCPullClientAuth'
Get-Certificate -Url https://$webenrollURL/ADPolicyProvider_CEP_UsernamePassword/service.svc/CEP `
                -Template $templateName `
                -SubjectName 'CN=TestEnroll' `
                -Credential $credentials `
                -CertStoreLocation Cert:\LocalMachine\My `
                -Verbose
```

As a result you should have acquired a certificate!

![](/images/2015-02/BG_VMRole_DSC_P02P030.png)

Run Certlm.msc to verify in the GUI.

![](/images/2015-02/BG_VMRole_DSC_P02P031.png)

That’s it for this post. Next up, the VM Role will be created handling the first automation activities:

* Root CA certificate download.
* Root CA cert import.
* Request Client Authentication Certificate.