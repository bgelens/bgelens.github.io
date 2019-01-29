---
title:  "Azure ARM Linked Templates and Complete mode"
date:   2019-01-29 12:00:00
categories: ["Azure","ARM"]
tags: ["Azure","ARM"]
excerpt_separator: <!--more-->
---

Just a small blog post on Azure ARM Linked Template deployment and Complete mode since I couldn't find a satisfying answer quickly enough via my favorite search engine.

>TL;DR: Linked Template resources together with the master template resources are deployed / kept / updated. Resources out of the cumulative result of master + linked templates are deleted.

<!--more-->

Today I was wondering what would happen if I had a "master" template being deployed in complete mode, what would happen if it had [Linked template](https://docs.microsoft.com/en-us/azure/azure-resource-manager/resource-group-linked-templates){:target="_blank"} deployments.

I have this sample "master" deployment template:

```json
{
    "$schema": "https://schema.management.azure.com/schemas/2015-01-01/deploymentTemplate.json#",
    "contentVersion": "1.0.0.0",
    "parameters": {},
    "variables": {},
    "resources": [
        {
            "type": "Microsoft.Storage/storageAccounts",
            "apiVersion": "2018-07-01",
            "name": "bgelensst01",
            "location": "[resourceGroup().location]",
            "sku": {
                "name":"Standard_LRS"
            }
        },
        {
            "name": "createsecond",
            "type": "Microsoft.Resources/deployments",
            "apiVersion": "2015-01-01",
            "properties": {
                "mode": "Incremental",
                "templateLink": {
                    "uri": "https://boguslink/storage.json",
                    "contentVersion": "1.0.0.0"
                }
            }
        }
    ],
    "outputs": {}
}
```

>Note that it is only supported to have `Incremental` mode specified for Linked deployements.

The linked storage.json looks like this:

```json
{
    "$schema": "https://schema.management.azure.com/schemas/2015-01-01/deploymentTemplate.json#",
    "contentVersion": "1.0.0.0",
    "parameters": {},
    "variables": {},
    "resources": [
        {
            "type": "Microsoft.Storage/storageAccounts",
            "apiVersion": "2018-07-01",
            "name": "bgelensst02",
            "location": "[resourceGroup().location]",
            "sku": {
                "name":"Standard_LRS"
            }
        }
    ],
    "outputs": {}
}
```

I created a resource group called `complete` and deployed the template to it using PowerShell:

```powershell
New-AzResourceGroupDeployment -ResourceGroupName complete -TemplateFile .\azuredeploy.1.json -Mode Complete -Force
```

![completedeploy01](/images/2019-01/completedeploy01.png)

The result is 2 Storage Accounts. So no problem there even though you might think to have a conflicting situation with the Linked deployment being in incremental mode.

![resultcomplete01](/images/2019-01/resultcomplete01.png)

Now let's add another resource to the resource group which is not specified in one of templates.

![completewithnsg](/images/2019-01/completewithnsg.png)

I've added a NSG. Let's do another deployment of the master template in complete mode and check the result.

```powershell
New-AzResourceGroupDeployment -ResourceGroupName complete -TemplateFile .\azuredeploy.1.json -Mode Complete -Force
```

The NSG is removed but the storage accounts remained. When checking the related events we find the delete operation of the NSG.

![completeactivity](/images/2019-01/completeactivity.png)

To summarize, Linked Template resources together with the master template resources are deployed / kept / updated. Resources out of the cumulative result of master + linked templates are deleted.
