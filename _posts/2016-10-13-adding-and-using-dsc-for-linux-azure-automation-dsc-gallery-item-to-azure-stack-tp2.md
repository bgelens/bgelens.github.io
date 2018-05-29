---
title:  "Adding and using DSC For Linux Azure Automation DSC Gallery Item to Azure Stack TP2"
date:   2016-10-13 12:00:00
categories: ["AzureStack","MAS","Linux","Desired State Configuration"]
tags: ["AzureStack","MAS","Linux","Desired State Configuration"]
---

This blog post is written and published on AzureStack.blog: [https://azurestack.blog/2016/10/adding-and-using-dsc-for-linux-azure-automation-dsc-gallery-item-to-azure-stack-tp2/](https://azurestack.blog/2016/10/adding-and-using-dsc-for-linux-azure-automation-dsc-gallery-item-to-azure-stack-tp2/){:target="_blank"}

In the previous post we added the DSC For Linux VM Extension Gallery item to Azure Stack and used it during deployments with a precompiled mof. In this post we will create another where the target is to onboard the Linux node into Azure Automation DSC.

In this series:

Adding and using…

* [CentOS 7.2 (or any other image) to Azure Stack TP2](https://bgelens.nl/adding-and-using-centos-7-2-or-any-other-image-to-azure-stack-tp2/)
* [OS Gallery Items to Azure Stack TP2](https://bgelens.nl/adding-and-using-os-gallery-items-to-azure-stack-tp2/)
* [The DSC For Linux extension v2 to Azure Stack TP2](https://bgelens.nl/adding-and-using-the-dsc-for-linux-extension-v2-to-azure-stack-tp2/)
* [DSC For Linux Gallery Item to Azure Stack TP2](https://bgelens.nl/adding-and-using-dsc-for-linux-gallery-item-to-azure-stack-tp2/)
* DSC For Linux Azure Automation DSC Gallery Item to Azure Stack TP2 (this post)

# VM Extension Gallery Item

In this blog post, we’ll create a Gallery item to expose the DSC For Linux VM extension in regard to the second use case (Register). As most of this was already covered in the previous post, we’ll dive right in. You can just copy your previous files into a new folder and start from there.

## Artifact structure

The artifact has the following structure and files:

![BG_AS_05CRPAddon_01](/images/2016-10/BG_AS_05CRPAddon_01.png)

We again have a root folder and 3 sub folders (DeploymentTemplates, icons and strings).

### Root level documents folder

At the root level we have 2 json files:

* Manifest.json
* UIDefinition.json

#### Manifest.json

The manifest for this Gallery Items differs from the previous one only for its name key which is now “DSCForLinuxAADSC-arm”.

```json
{
    "$schema": "https://gallery.azure.com/schemas/2015-10-01/manifest.json#",
    "name": "DSCForLinuxAADSC-arm",
    "publisher": "Microsoft",
    "version": "1.0.0",
    "displayName": "ms-resource:displayName",
    "publisherDisplayName": "ms-resource:publisherDisplayName",
    "publisherLegalName": "ms-resource:publisherLegalName",
    "summary": "ms-resource:summary",
    "longSummary": "ms-resource:longSummary",
    "description": "ms-resource:description",
    "uiDefinition": {
        "path": "UIDefinition.json"
    },
    "metadata": [],
    "artifacts": [
        {
            "name": "MainTemplate",
            "type": "Template",
            "path": "DeploymentTemplates\\MainTemplate.json",
            "isDefault": true
        },
        {
            "name": "CreateUiDefinition",
            "type": "Custom",
            "path": "DeploymentTemplates\\CreateUiDefinition.json",
            "isDefault": false
        }
    ],
    "properties": [],
    "categories": [
        "compute-vmextension-linux"
    ],
    "links": [
        {
            "displayName": "ms-resource:linkDisplayName0",
            "uri": "https://github.com/Azure/azure-linux-extensions/tree/master/DSC"
        }
    ],
    "filters": [],
    "products": [],
    "images": [
        {
            "context": "ibiza",
            "items": [
                {
                    "id": "small",
                    "path": "icons\\Small.png",
                    "type": "icon"
                },
                {
                    "id": "medium",
                    "path": "icons\\Medium.png",
                    "type": "icon"
                },
                {
                    "id": "large",
                    "path": "icons\\Large.png",
                    "type": "icon"
                },
                {
                    "id": "wide",
                    "path": "icons\\Wide.png",
                    "type": "icon"
                },
                {
                    "id": "screenshot1",
                    "path": "icons\\screenshot1.png",
                    "type": "screenshot"
                }
            ]
        }
    ]
}
```

#### UIDefinition.json

The UIDefinition.json is exactly the same. No change here.

```json
{
  "$schema": "https://gallery.azure.com/schemas/2015-02-12/uiDefinition.json#",
  "createDefinition": {
    "createBlade": {
      "name": "AddVmExtension",
      "extension": "Microsoft_Azure_Compute"
    }
  }
}
```

### DeploymentTemplates folder

The DeploymentTemplates folder contains two json files which do differ a bit:

* CreateUiDefinition.json
* MainTemplate.json

#### CreateUiDefinition.json

CreateUiDefinition is tuned to the register use case.

```json
{
  "handler": "Microsoft.Compute.VmExtension",
  "version": "0.0.1-preview",
  "parameters": {
    "elements": [
      {
        "name": "RegistrationUrl",
        "type": "Microsoft.Common.TextBox",
        "label": "URL of Azure Automation DSC Account",
        "toolTip": "enter the URL of your automation account",
        "constraints": {
          "required": true
        }
      },
      {
        "name": "RegistrationKey",
        "type": "Microsoft.Common.PasswordBox",
        "label": "Primary or secondary key of Azure Automation DSC Account",
        "toolTip": "enter the Primary or secondary key of your automation account",
        "constraints": {
          "required": true
        }
      }
    ],
    "outputs": {
      "vmName": "[vmName()]",
      "location": "[location()]",
      "RegistrationUrl": "[elements('RegistrationUrl')]",
      "RegistrationKey": "[elements('RegistrationKey')]"
    }
  }
}
```

In this case I have defined 2 elements:

* RegistrationUrl
* RegistrationKey

##### RegistrationUrl and RegistrationKey

```json
{
    "name": "RegistrationUrl",
    "type": "Microsoft.Common.TextBox",
    "label": "URL of Azure Automation DSC Account",
    "toolTip": "enter the URL of your automation account",
    "constraints": {
        "required": true
    }
},
{
    "name": "RegistrationKey",
    "type": "Microsoft.Common.TextBox",
    "label": "Primary or secondary key of Azure Automation DSC Account",
    "toolTip": "enter the Primary or secondary key of your automation account",
    "constraints": {
        "required": true
    }
}
```

The type of these elements is “Microsoft.Common.TextBox”, which is generated by the Portal as a free text field.

![BG_AS_05CRPAddon_02](/images/2016-10/BG_AS_05CRPAddon_02.png)

You can see the labels showing up and the tooltips when clicked / hovered over, will show the info we provided.

![BG_AS_05CRPAddon_03](/images/2016-10/BG_AS_05CRPAddon_03.png)

##### Outputs

At the end of the CreateUiDefinition.json, outputs are specified. These outputs are used to send as values for the parameters of MainTemplate.json which is used by the Portal to add the VM Extension to an already deployed VM or merge into the overall JSON generated for a VM from scratch deployment.

```json
"outputs": {
    "vmName": "[vmName()]",
    "location": "[location()]",
    "RegistrationUrl": "[elements('RegistrationUrl')]",
    "RegistrationKey": "[elements('RegistrationKey')]"
}
```

#### MainTemplate.json

MainTemplate again, is basically a JSON template you would create yourself to add the DSC For Linux VM Extension to an already deployed VM.

```json
{
  "$schema": "http://schema.management.azure.com/schemas/2015-01-01/deploymentTemplate.json",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "vmName": {
      "type": "string"
    },
    "location": {
      "type": "string"
    },
    "RegistrationUrl": {
      "type": "string"
    },
    "RegistrationKey": {
      "type": "string"
    }
  },
  "resources": [
    {
      "name": "[concat(parameters('vmName'),'/Microsoft.OSTCExtensions.DSCForLinux')]",
      "type": "Microsoft.Compute/virtualMachines/extensions",
      "location": "[parameters('location')]",
      "apiVersion": "2015-06-15",
      "properties": {
        "publisher": "Microsoft.OSTCExtensions",
        "type": "DSCForLinux",
        "typeHandlerVersion": "2.0",
        "autoUpgradeMinorVersion": false,
        "settings": {
          "Mode": "Register"
        },
        "protectedSettings": {
          "RegistrationUrl": "[parameters('RegistrationUrl')]",
          "RegistrationKey": "[parameters('RegistrationKey')]"
        }
      }
    }
  ]
}
```

I created parameters which correspond with the outputs from the CreateUiDefinition.json and mapped those to the resource properties where applicable. As you can see this template only has the public settings Mode which is hardcoded to Register. Besided the public settings, protectedSettings are defined in this case for the registration url and key.

### Icons

I just kept the same icons from the previous blog.

Name|px|image
----|--|-----
Large.png|115 x 115|![BG_AS_04CRPAddon_09](/images/2016-10/BG_AS_04CRPAddon_09.png)
Medium.png|90 x 90|![BG_AS_04CRPAddon_10](/images/2016-10/BG_AS_04CRPAddon_10.png)
Small.png|40 x 40|![BG_AS_04CRPAddon_11](/images/2016-10/BG_AS_04CRPAddon_11.png)
Wide.png|255 x 115|![BG_AS_04CRPAddon_12](/images/2016-10/BG_AS_04CRPAddon_12.png)
Screenshot1.png|533 x 324|![BG_AS_04CRPAddon_13](/images/2016-10/BG_AS_04CRPAddon_13.png)

### Strings

The strings from strings\resources.resjson are used by the manifest file.

```json
{
    "displayName": "AA DSC For Linux",
    "publisherDisplayName": "Ben Gelens.",
    "publisherLegalName": "Ben Gelens.",
    "summary": "Onboard Linux into Azure Automation DSC",
    "longSummary": "Onboard Linux into Azure Automation DSC",
    "description": "<p>DSC is a management platform that enables deploying and managing configuration data for software services and managing the environment in which these services run.</p>",
    "linkDisplayName0": "Learn more about DSC for Linux"
}
```

## Packaging Gallery Item

Let’s move the folder containing the files to the root of C. Now open a command prompt or PowerShell console and navigate to the packager and run: .\AzureGalleryPackageGenerator\AzureGalleryPackager.exe -m C:\ Microsoft.OSTCExtensions.DSCForLinuxAADSC-arm.1.0.0\Manifest.json -o C:\MyMarketPlaceItems

You should now have a marketplace item in azpkg format. If the packager did not work out, you probably have made some sort of typo somewhere or it just doesn’t like the paths you provide. In any case, you can download the azpkg I created [here](https://raw.githubusercontent.com/bgelens/BlogItems/master/Microsoft.DSCForLinuxAADSC-arm.1.0.0.azpkg){:target="_blank"}.

## Adding the Gallery Item to Azure Stack

We are going to use the same storage account again from the first blog post, if you did not create it, please go back to that one and create it.

Run the following script to add the azpkg to Azure Stack (I assume you already setup the Azure Stack connection in PowerShell).

```powershell
$subscriptionid = (Get-AzureRmSubscription -SubscriptionName 'Default Provider Subscription').SubscriptionId
$StorageAccount = Get-AzureRmStorageAccount -ResourceGroupName tenantartifacts -Name tenantartifacts
$GalleryContainer = Get-AzureStorageContainer -Name gallery -Context $StorageAccount.Context
$DSCazpkg = $GalleryContainer | Set-AzureStorageBlobContent -File C:\MyMarketPlaceItems\Microsoft.DSCForLinuxAADSC-arm.1.0.0.azpkg 
Add-AzureRMGalleryItem -SubscriptionId $subscriptionid -GalleryItemUri $DSCazpkg.ICloudBlob.StorageUri.PrimaryUri.AbsoluteUri  -Apiversion "2015-04-01"
```

You should see a message with StatusCode Ok.

![BG_AS_05CRPAddon_09](/images/2016-10/BG_AS_05CRPAddon_09.png)

Now we login to the portal, wait a bit, refresh a couple of times and eventually we get:

![BG_AS_05CRPAddon_10](/images/2016-10/BG_AS_05CRPAddon_10.png)

Note, if you made an error somewhere (wrong text or something) and want to re-add the Gallery item. You should remove it first.

```powershell
Remove-AzureRMGalleryItem -Name 'Microsoft.DSCForLinux-arm.1.0.0' -ApiVersion 2015-04-01
```

## Using the Gallery Item

Now we made the Gallery Item available, let’s put it to work. You need the Uri and one of the access keys from your Azure Automation account. Let’s start to create a new VM based on our CentOS 7.2 Gallery Item.

At step 3 select the Extensions blade and then Add Extension.

![BG_AS_05CRPAddon_11](/images/2016-10/BG_AS_05CRPAddon_11.png)

Now we should see the DSC For Linux VM Extension Gallery Item.

![BG_AS_05CRPAddon_12](/images/2016-10/BG_AS_05CRPAddon_12.png)

Select it and hit create.

![BG_AS_05CRPAddon_13](/images/2016-10/BG_AS_05CRPAddon_13.png)

We can now see the fields we made available. Let’s provide it with the Azure Automation Account Uri and key.

![BG_AS_05CRPAddon_14](/images/2016-10/BG_AS_05CRPAddon_14.png)

Now we have the extension configured, proceed as normal and deploy the VM. Once finished we the node should be onboarded into Azure Automation DSC.

![BG_AS_05CRPAddon_15](/images/2016-10/BG_AS_05CRPAddon_15.png)

That’s it for this blog series for now. I do have some follow up ideas however, so probably be expanding this series in the near future!
