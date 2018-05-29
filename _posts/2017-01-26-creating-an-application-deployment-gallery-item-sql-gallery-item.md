---
title:  "Creating an Application Deployment Gallery Item – SQL Gallery item"
date:   2017-01-26 12:00:00
categories: ["AzureStack","MAS","Linux","Desired State Configuration"]
tags: ["AzureStack","MAS","Linux","Desired State Configuration"]
---

This blog post is written and published on AzureStack.blog: [https://azurestack.blog/2017/01/creating-an-application-deployment-gallery-item-sql-gallery-item/](https://azurestack.blog/2017/01/creating-an-application-deployment-gallery-item-sql-gallery-item/){:target="_blank"}

Welcome to the second blog post (of 2) on creating a complex application deployment gallery item.

* [Part 1 – Prerequisites](https://bgelens.nl/creating-an-application-deployment-gallery-item-prerequisites/)
* Part 2 – SQL Gallery item (this post)

Now we got all fundamentals in place, we need a deployment template to thigh everything together into a deployable artefact and expose that to the portal via a Gallery Item.

Microsoft made a template available on GitHub to get you started with creating this complex gallery item: [https://github.com/Azure/azure-quickstart-templates/tree/master/marketplace-samples/simple-windows-vm](https://github.com/Azure/azure-quickstart-templates/tree/master/marketplace-samples/simple-windows-vm){:target="_blank"}

For this example, we keep things simple just to prove what can be done. A lot more flexibility can be achieved by utilizing nested templates in combination with option controls for example.

Let’s create the Gallery Item!

## Directory structure

For this case, we’ll use the directory structure as displayed in the image.

![BG_AS_02_ComplexVMHandler_01](/images/2017-01/BG_AS_02_ComplexVMHandler_01.png)

The directory structure is in no means mandatory as files are referenced using relative paths.

### Deployment Template

The Gallery Item in its core is a UI wrapper around a deployment template. The deployment template will be stored in the DeploymentTemplates folder and will be called “mainTemplate.json”. Current experience tells that the filename of this artefact cannot be any different, if you don’t use this name, things just won’t work. The template is declarative and speaks largely for itself so I won’t go into detail about it.

![BG_AS_02_ComplexVMHandler_02](/images/2017-01/BG_AS_02_ComplexVMHandler_02.png)

Please note the parameter mapping to the DSC extension where the SQLPID and username / password are send to the secure settings so their values won’t be visible at the resource group deployment history when the deployment is done.

```json
{
    "$schema": "https://schema.management.azure.com/schemas/2015-01-01/deploymentTemplate.json#",
    "contentVersion": "1.0.0.0",
    "parameters": {
        "adminUsername": {
            "type": "string",
            "metadata": {
                "description": "Username for the Virtual Machine."
            }
        },
        "adminPassword": {
            "type": "securestring",
            "metadata": {
                "description": "Password for the Virtual Machine."
            }
        },
        "vmSize": {
            "type": "string",
            "allowedValues": [
                "Basic_A0",
                "Basic_A1",
                "Basic_A2",
                "Basic_A3",
                "Basic_A4",
                "Standard_A0",
                "Standard_A1",
                "Standard_A2",
                "Standard_A3",
                "Standard_A4"
            ],
            "defaultValue": "Standard_A2"
        },
        "vmName": {
            "type": "string",
            "maxLength": 15,
            "minLength": 3,
            "defaultValue": "MySQLVM"
        },
        "sqlPort": {
            "type": "int",
            "defaultValue": 1433,
            "metadata": {
                "description": "SQL instance port. Port will be opened through NSG."
            }
        },
        "sqlInstanceName": {
            "type": "string",
            "minLength": 1,
            "maxLength": 15,
            "defaultValue": "MSSQLSERVER",
            "metadata": {
                "description": "SQL instance name. MSSQLSERVER will create default instance."
            }
        },
        "sqlDataDiskSize": {
            "type": "int",
            "defaultValue": "100",
            "minValue": 10,
            "maxValue": 200,
            "metadata": {
                "description": "Specify the size of the SQL data disk."
            }
        },
        "sqlLogDiskSize": {
            "type": "int",
            "defaultValue": "100",
            "minValue": 10,
            "maxValue": 200,
            "metadata": {
                "description": "Specify the size of the SQL log disk."
            }
        },
        "sqlFeatures": {
            "type": "string",
            "allowedValues": [
                "SQLENGINE",
                "IS",
                "SQLENGINE, IS"
            ],
            "metadata": {
                "description": "Specify the SQL Features you want to install."
            },
            "defaultValue": "SQLENGINE"
        },
        "sqlAuthenticationMode": {
            "type": "string",
            "allowedValues": [
                "SQL",
                "Windows"
            ],
            "metadata": {
                "description": "Specify the SQL Authentication mode you want to use for the SQLENGINE feature."
            },
            "defaultValue": "SQL"
        },
        "sqlPID": {
            "defaultValue": "",
            "type": "string"
        }
    },
    "variables": {
        "storageAccountName": "[concat(uniquestring(resourceGroup().id), 'sqlvm')]",
        "nicName": "[concat(parameters('vmName'), '_nic')]",
        "addressPrefix": "10.0.0.0/16",
        "subnetName": "Subnet",
        "subnetPrefix": "10.0.0.0/24",
        "publicIPAddressName": "[concat(parameters('vmName'), '_pip')]",
        "virtualNetworkName": "[concat(parameters('vmName'), '_vnet')]",
        "subnetRef": "[resourceId('Microsoft.Network/virtualNetworks/subnets', variables('virtualNetworkName'), variables('subnetName'))]",
        "networkSecurityGroupName": "[concat(parameters('vmName'), '_nsg')]",
        "sqlFeatures": "[split(parameters('sqlFeatures'), ',')]",
        "imagePublisher": "Microsoft",
        "imageOffer": "WindowsServer",
        "imageSku": "2016-Standard-Core"
    },
    "resources": [
        {
            "type": "Microsoft.Storage/storageAccounts",
            "name": "[variables('storageAccountName')]",
            "apiVersion": "2016-01-01",
            "location": "[resourceGroup().location]",
            "sku": {
                "name": "Standard_LRS"
            },
            "kind": "Storage",
            "properties": {}
        },
        {
            "apiVersion": "2015-06-15",
            "type": "Microsoft.Network/publicIPAddresses",
            "name": "[variables('publicIPAddressName')]",
            "location": "[resourceGroup().location]",
            "properties": {
                "publicIPAllocationMethod": "Dynamic"
            }
        },
        {
            "apiVersion": "2015-06-15",
            "type": "Microsoft.Network/virtualNetworks",
            "name": "[variables('virtualNetworkName')]",
            "location": "[resourceGroup().location]",
            "properties": {
                "addressSpace": {
                    "addressPrefixes": [
                        "[variables('addressPrefix')]"
                    ]
                },
                "subnets": [
                    {
                        "name": "[variables('subnetName')]",
                        "properties": {
                            "addressPrefix": "[variables('subnetPrefix')]"
                        }
                    }
                ]
            }
        },
        {
            "name": "[variables('networkSecurityGroupName')]",
            "type": "Microsoft.Network/networkSecurityGroups",
            "apiVersion": "2015-06-15",
            "location": "[resourceGroup().location]",
            "properties": {
                "securityRules": [
                    {
                        "name": "default-allow-rdp",
                        "properties": {
                            "priority": 1000,
                            "sourceAddressPrefix": "*",
                            "protocol": "Tcp",
                            "destinationPortRange": "3389",
                            "access": "Allow",
                            "direction": "Inbound",
                            "sourcePortRange": "*",
                            "destinationAddressPrefix": "*"
                        }
                    },
                    {
                        "name": "default-allow-winrm",
                        "properties": {
                            "priority": 1100,
                            "sourceAddressPrefix": "*",
                            "protocol": "Tcp",
                            "destinationPortRange": "5985",
                            "access": "Allow",
                            "direction": "Inbound",
                            "sourcePortRange": "*",
                            "destinationAddressPrefix": "*"
                        }
                    },
                    {
                        "name": "default-allow-winrm-ssl",
                        "properties": {
                            "priority": 1200,
                            "sourceAddressPrefix": "*",
                            "protocol": "Tcp",
                            "destinationPortRange": "5986",
                            "access": "Allow",
                            "direction": "Inbound",
                            "sourcePortRange": "*",
                            "destinationAddressPrefix": "*"
                        }
                    },
                    {
                        "name": "default-allow-sql",
                        "properties": {
                            "priority": 1300,
                            "sourceAddressPrefix": "*",
                            "protocol": "Tcp",
                            "destinationPortRange": "[parameters('sqlPort')]",
                            "access": "Allow",
                            "direction": "Inbound",
                            "sourcePortRange": "*",
                            "destinationAddressPrefix": "*"
                        }
                    }
                ]
            }
        },
        {
            "apiVersion": "2015-06-15",
            "type": "Microsoft.Network/networkInterfaces",
            "name": "[variables('nicName')]",
            "location": "[resourceGroup().location]",
            "dependsOn": [
                "[resourceId('Microsoft.Network/publicIPAddresses/', variables('publicIPAddressName'))]",
                "[resourceId('Microsoft.Network/virtualNetworks/', variables('virtualNetworkName'))]",
                "[resourceId('Microsoft.Network/networkSecurityGroups/', variables('networkSecurityGroupName'))]"
            ],
            "properties": {
                "ipConfigurations": [
                    {
                        "name": "ipconfig1",
                        "properties": {
                            "privateIPAllocationMethod": "Dynamic",
                            "publicIPAddress": {
                                "id": "[resourceId('Microsoft.Network/publicIPAddresses',variables('publicIPAddressName'))]"
                            },
                            "subnet": {
                                "id": "[variables('subnetRef')]"
                            }
                        }
                    }
                ],
                "networkSecurityGroup": {
                    "id": "[resourceId('Microsoft.Network/networkSecurityGroups/', variables('networkSecurityGroupName'))]"
                }
            }
        },
        {
            "apiVersion": "2015-06-15",
            "type": "Microsoft.Compute/virtualMachines",
            "name": "[parameters('vmName')]",
            "location": "[resourceGroup().location]",
            "dependsOn": [
                "[resourceId('Microsoft.Storage/storageAccounts/', variables('storageAccountName'))]",
                "[resourceId('Microsoft.Network/networkInterfaces/', variables('nicName'))]"
            ],
            "properties": {
                "hardwareProfile": {
                    "vmSize": "[parameters('VMSize')]"
                },
                "osProfile": {
                    "computerName": "[parameters('vmName')]",
                    "adminUsername": "[parameters('adminUsername')]",
                    "adminPassword": "[parameters('adminPassword')]"
                },
                "storageProfile": {
                    "imageReference": {
                        "publisher": "[variables('imagePublisher')]",
                        "offer": "[variables('imageOffer')]",
                        "sku": "[variables('imageSKU')]",
                        "version": "latest"
                    },
                    "osDisk": {
                        "name": "osdisk",
                        "vhd": {
                            "uri": "[concat(reference(resourceId('Microsoft.Storage/storageAccounts/', variables('storageAccountName'))).primaryEndpoints.blob, 'vhds/osdisk.vhd')]"
                        },
                        "caching": "ReadWrite",
                        "createOption": "FromImage"
                    },
                    "dataDisks": [
                        {
                            "createOption": "Empty",
                            "lun": 0,
                            "name": "[concat(parameters('vmName'), '_sqldata')]",
                            "diskSizeGB": "[parameters('sqlDataDiskSize')]",
                            "vhd": {
                                "uri": "[concat(reference(resourceId('Microsoft.Storage/storageAccounts/', variables('storageAccountName'))).primaryEndpoints.blob, parameters('vmName'), '_sqldata.vhd')]"
                            }
                        },
                        {
                            "createOption": "Empty",
                            "lun": 1,
                            "name": "[concat(parameters('vmName'), '_sqllog')]",
                            "diskSizeGB": "[parameters('sqlLogDiskSize')]",
                            "vhd": {
                                "uri": "[concat(reference(resourceId('Microsoft.Storage/storageAccounts/', variables('storageAccountName'))).primaryEndpoints.blob, parameters('vmName'), '_sqllog.vhd')]"
                            }
                        }
                    ]
                },
                "networkProfile": {
                    "networkInterfaces": [
                        {
                            "id": "[resourceId('Microsoft.Network/networkInterfaces',variables('nicName'))]"
                        }
                    ]
                }
            }
        },
        {
            "type": "Microsoft.Compute/virtualMachines/extensions",
            "name": "[concat(parameters('vmName'),'/dscExtension')]",
            "apiVersion": "2015-06-15",
            "properties": {
                "autoUpgradeMinorVersion": true,
                "publisher": "Microsoft.Powershell",
                "type": "DSC",
                "typeHandlerVersion": "2.19",
                "protectedSettings": {
                    "configurationArguments": {
                        "SetupCredentials": {
                            "userName": "[parameters('adminUsername')]",
                            "password": "[parameters('adminPassword')]"
                        },
                        "ProductId": "[parameters('sqlPID')]"
                    }
                },
                "settings": {
                    "wmfVersion": "latest",
                    "privacy": {
                        "dataCollection": "disable"
                    },
                    "configuration": {
                        "url": "https://tenantartifacts.blob.azurestack.local/dsc/SQLConfiguration.ps1.zip",
                        "script": "SQLConfiguration.ps1",
                        "function": "SQLConfiguration"
                    },
                    "configurationArguments": {
                        "SQLInstanceName": "[parameters('sqlInstanceName')]",
                        "Features": "[variables('sqlFeatures')]",
                        "Port": "[parameters('sqlPort')]",
                        "SecurityMode": "[parameters('sqlAuthenticationMode')]"
                    }
                }
            },
            "dependsOn": [
                "[resourceId('Microsoft.Compute/virtualMachines/', parameters('vmName'))]"
            ]
        }
    ],
    "outputs": {}
}
```

### CreateUIDefinition

The UI in the portal is generated from the CreateUIDefinition.json. The portal exposes a plural off controls which can be leveraged. The main goal is to gather the user input for the deployment template to consume. The main outline of the CreateUIDefinition.json looks like this:

![BG_AS_02_ComplexVMHandler_03](/images/2017-01/BG_AS_02_ComplexVMHandler_03.png)

It is possible to add a schema reference during editing to help you with intellisense like completions, the schema is currently the best documentation out there to know what is possible, so take a look [here](https://github.com/Azure/azure-resource-manager-schemas/tree/master/schemas/0.1.2-preview){:target="_blank"} if you are interested in more than documented here).

![BG_AS_02_ComplexVMHandler_04](/images/2017-01/BG_AS_02_ComplexVMHandler_04.png)

First let’s view the CreateUIDefinition and then explain the controls that are used and how they are rendered. The CreateUIDefinition.json is also stored inside the DeploymentTemplates folder.

```json
{
    "$schema": "https://schema.management.azure.com/schemas/0.1.2-preview/CreateUIDefinition.MultiVm.json",
    "handler": "Microsoft.Compute.MultiVm",
    "version": "0.1.2-preview",
    "parameters": {
        "basics": [
            {
                "name": "vmName",
                "label": "Name",
                "type": "Microsoft.Common.TextBox",
                "constraints": {
                    "required": true,
                    "regex": "\\b\\w{3,15}\\b",
                    "validationMessage": "Name lenght must be at least 3 and maximum 15 characters"
                },
                "defaultValue": "MySQLVM",
                "toolTip": "ComputerName for the VM."
            },
            {
                "name": "adminUsername",
                "type": "Microsoft.Compute.UserNameTextBox",
                "label": "Username",
                "toolTip": "Admin username for the virtual machine.",
                "osPlatform": "Windows"
            },
            {
                "name": "adminPassword",
                "type": "Microsoft.Compute.CredentialsCombo",
                "label": {
                    "password": "Password",
                    "confirmPassword": "Confirm password"
                },
                "toolTip": {
                    "password": "Admin password for the virtual machine."
                },
                "osPlatform": "Windows",
                "options": {
                    "hideConfirmation": false
                }
            }
        ],
        "steps": [
            {
                "name": "resourceCustomization",
                "label": "Resource Customization",
                "subLabel": {
                    "preValidation": "Configure Resource Customization",
                    "postValidation": "Done"
                },
                "bladeTitle": "Resource Customization",
                "bladeSubtitle": "Configure Resource Customization",
                "elements": [
                    {
                        "type": "Microsoft.Compute.SizeSelector",
                        "name": "vmSize",
                        "label": "Virtual Machine Size",
                        "count": 1,
                        "osPlatform": "Windows",
                        "recommendedSizes": [
                            "Standard_A2",
                            "Standard_A3",
                            "Standard_A4"
                        ],
                        "constraints": {
                            "allowedSizes": [
                                "Standard_A0",
                                "Standard_A1",
                                "Standard_A2",
                                "Standard_A3",
                                "Standard_A4"
                            ]
                        },
                        "imageReference": {
                            "offer": "WindowsServer",
                            "publisher": "Microsoft",
                            "sku": "2016-Standard-Core"
                        }
                    },
                    {
                        "type": "Microsoft.Common.TextBox",
                        "constraints": {
                            "regex": "^\\d+$",
                            "validationMessage": "Value must be an integer.",
                            "required": true
                        },
                        "defaultValue": "100",
                        "name": "sqlDataDiskSize",
                        "label": "Provide SQL Data Disk size in GB."
                    },
                    {
                        "type": "Microsoft.Common.TextBox",
                        "constraints": {
                            "regex": "^\\d+$",
                            "validationMessage": "Value must be an integer.",
                            "required": true
                        },
                        "defaultValue": "50",
                        "name": "sqlLogDiskSize",
                        "label": "Provide SQL Log Disk size in GB."
                    }
                ]
            },
            {
                "name": "sqlSettings",
                "label": "SQL Settings",
                "subLabel": {
                    "preValidation": "Configure SQL Settings",
                    "postValidation": "Done"
                },
                "bladeTitle": "SQL Settings",
                "bladeSubtitle": "Configure SQL Settings",
                "elements": [
                    {
                        "type": "Microsoft.Common.TextBox",
                        "constraints": {
                            "required": false
                        },
                        "label": "SQL Product key",
                        "name": "sqlProductKey",
                        "toolTip": "if you want to use the trial, leave empty.",
                        "defaultValue": ""
                    },
                    {
                        "type": "Microsoft.Common.DropDown",
                        "name": "sqlFeatures",
                        "label": "Select SQL Features",
                        "constraints": {
                            "allowedValues": [
                                {
                                    "label": "SQL Engine",
                                    "value": "SQLENGINE"
                                },
                                {
                                    "label": "Integration Services",
                                    "value": "IS"
                                },
                                {
                                    "label": "SQL Engine + IS",
                                    "value": "SQLENGINE, IS"
                                }
                            ],
                            "required": true
                        }
                    },
                    {
                        "type": "Microsoft.Common.Section",
                        "name": "sqlEngine",
                        "label": "SQL Engine",
                        "visible": "[contains(steps('sqlSettings').sqlFeatures, 'SQLENGINE')]",
                        "elements": [
                            {
                                "type": "Microsoft.Common.TextBox",
                                "name": "sqlInstanceName",
                                "label": "Instance name",
                                "defaultValue": "MSSQLSERVER",
                                "constraints": {
                                    "required": true,
                                    "regex": "(?!^[dD][eE][fF][aA][uU][lL][tT]$)(^[a-zA-Z_][0-9a-zA-Z$_]{1,15}$)",
                                    "validationMessage": "Instance name must comply with Instance name rules."
                                },
                                "toolTip": "1-16 chars, can start with '_', cannot contain illigal characters (!, #, :, ;), cannot be named 'Default'"
                            },
                            {
                                "type": "Microsoft.Common.TextBox",
                                "constraints": {
                                    "required": true,
                                    "regex": "^\\d+$",
                                    "validationMessage": "Value must be an integer."
                                },
                                "defaultValue": "1433",
                                "name": "sqlInstancePort",
                                "label": "Instance Port",
                                "toolTip": "Default port is 1433"
                            },
                            {
                                "type": "Microsoft.Common.OptionsGroup",
                                "constraints": {
                                    "allowedValues": [
                                        {
                                            "label": "SQL",
                                            "value": "SQL"
                                        },
                                        {
                                            "label": "Windows",
                                            "value": "Windows"
                                        }
                                    ],
                                    "required": true
                                },
                                "defaultValue": "SQL",
                                "label": "Authentication Mode",
                                "name": "sqlAuthMode",
                                "toolTip": "When selecting Windows, you might not be able to remote connect SSMS until the VM is joined to a domain."
                            }
                        ]
                    }
                ]
            }
        ],
        "outputs": {
            "adminUsername": "[basics('adminUsername')]",
            "adminPassword": "[basics('adminPassword').password]",
            "vmName": "[basics('vmName')]",
            "vmSize": "[steps('resourceCustomization').vmSize]",
            "sqlPort": "[int(steps('sqlSettings').sqlEngine.sqlInstancePort)]",
            "sqlInstanceName": "[steps('sqlSettings').sqlEngine.sqlInstanceName]",
            "sqlDataDiskSize": "[int(steps('resourceCustomization').sqlDataDiskSize)]",
            "sqlLogDiskSize": "[int(steps('resourceCustomization').sqlLogDiskSize)]",
            "sqlFeatures": "[steps('sqlSettings').sqlFeatures]",
            "sqlAuthenticationMode": "[steps('sqlSettings').sqlEngine.sqlAuthMode]",
            "sqlPID": "[steps('sqlSettings').sqlProductKey]"
        }
    }
}
```

The CreateUIDefinition rendering can be previewed before you decide to package it up by having the portal gather it externally. Use the following code to do this (I assume you already are connect with your Azure Stack environment using a Service Admin account, if you are not, see [here](https://docs.microsoft.com/en-us/azure/azure-stack/azure-stack-connect-powershell){:target="_blank"} how this is done. I also assume you created the “tenantartifacts” storage account in the previous blog post):

```powershell
$StorageAccount = Get-AzureRmStorageAccount -ResourceGroupName tenantartifacts -Name tenantartifacts
$CreateUIPreviewContainer = $StorageAccount | New-AzureStorageContainer -Name preview -Permission Blob
$Upload = $CreateUIPreviewContainer | Set-AzureStorageBlobContent -File (Resolve-Path ~\Desktop\CreateUIDefinition.json) -Force
$uiUriEscaped = [uri]::EscapeDataString($Upload.ICloudBlob.Uri)
$sideloadUri = "https://portal.azurestack.local/#blade/Microsoft_Azure_Compute/CreateMultiVmWizardBlade/internal_bladeCallId/anything/internal_bladeCallerParams/{`"initialData`":{},`"providerConfig`":{`"createUiDefinition`":`"$uiUriEscaped`"}}"
Start-Process $sideloadUri
```

This will upload the json to a storage account and construct a uri which tells the portal to load the json and display the content.

![BG_AS_02_ComplexVMHandler_05](/images/2017-01/BG_AS_02_ComplexVMHandler_05.png)

Now let’s look at how this json is build up.

#### Basics

Basics is a mandatory blade which always exist. Even when you decide not to populate it with controls, it is still used for things like Resource Group creation, location and subscription selection.

![BG_AS_02_ComplexVMHandler_06](/images/2017-01/BG_AS_02_ComplexVMHandler_06.png)

In our case, we decided to populate the basics blade with input parameters:

```json
"basics": [
    {
        "name": "vmName",
        "label": "Name",
        "type": "Microsoft.Common.TextBox",
        "constraints": {
            "required": true,
            "regex": "\\b\\w{3,15}\\b",
            "validationMessage": "Name lenght must be at least 3 and maximum 15 characters"
        },
        "defaultValue": "MySQLVM",
        "toolTip": "ComputerName for the VM."
    },
    {
        "name": "adminUsername",
        "type": "Microsoft.Compute.UserNameTextBox",
        "label": "Username",
        "toolTip": "Admin username for the virtual machine.",
        "osPlatform": "Windows"
    },
    {
        "name": "adminPassword",
        "type": "Microsoft.Compute.CredentialsCombo",
        "label": {
            "password": "Password",
            "confirmPassword": "Confirm password"
        },
        "toolTip": {
            "password": "Admin password for the virtual machine."
        },
        "osPlatform": "Windows",
        "options": {
            "hideConfirmation": false
        }
    }
]
```

Which is rendered as:

![BG_AS_02_ComplexVMHandler_07](/images/2017-01/BG_AS_02_ComplexVMHandler_07.png)

The input parameters:

* vmName: uses the Microsoft.Common.Textbox type. Input is required and the value is constrained to a pattern match with a defined regex (3 to 15 characters). The input is used for ComputerName and VMName.

constraint|effect
----------|------
Between 3 to 15 chars:|![BG_AS_02_ComplexVMHandler_08](/images/2017-01/BG_AS_02_ComplexVMHandler_08.png)
Under 3 chars:|![BG_AS_02_ComplexVMHandler_09](/images/2017-01/BG_AS_02_ComplexVMHandler_09.png)
Over 15 chars:|![BG_AS_02_ComplexVMHandler_10](/images/2017-01/BG_AS_02_ComplexVMHandler_10.png)

* adminUsername: uses the Microsoft.Compute.UserNameTextBox type. It is targeted for a Windows deployment and used for the Administrator user and potentially SQL login account (if the SQL authentication mode is set to SQL). The input is checked for illegal entries by portal native defined patterns.

constraint|effect
----------|------
Cannot contain illegal characters:|![BG_AS_02_ComplexVMHandler_11](/images/2017-01/BG_AS_02_ComplexVMHandler_11.png)
Between 1 and 15 chars:|![BG_AS_02_ComplexVMHandler_12](/images/2017-01/BG_AS_02_ComplexVMHandler_12.png)
Cannot be a single digit:|![BG_AS_02_ComplexVMHandler_13](/images/2017-01/BG_AS_02_ComplexVMHandler_13.png)
Can be a number over 1 digit:|![BG_AS_02_ComplexVMHandler_14](/images/2017-01/BG_AS_02_ComplexVMHandler_14.png)
Can be a number followed by a letter:|![BG_AS_02_ComplexVMHandler_15](/images/2017-01/BG_AS_02_ComplexVMHandler_15.png)
Can be a normal username:|![BG_AS_02_ComplexVMHandler_16](/images/2017-01/BG_AS_02_ComplexVMHandler_16.png)

* adminPassword: uses the Microsoft.Compute.CredentialsCombo type. Length and complexity is guarded by the portal, input is validated but can be hidden by setting hideConfirmation to true.

constraint|effect
----------|------
Between 12 and 123 characters|![BG_AS_02_ComplexVMHandler_17](/images/2017-01/BG_AS_02_ComplexVMHandler_17.png)
Complexity requirements:|![BG_AS_02_ComplexVMHandler_18](/images/2017-01/BG_AS_02_ComplexVMHandler_18.png)
Valid length:|![BG_AS_02_ComplexVMHandler_19](/images/2017-01/BG_AS_02_ComplexVMHandler_19.png)
Valid length, invalid confirmation:|![BG_AS_02_ComplexVMHandler_20](/images/2017-01/BG_AS_02_ComplexVMHandler_20.png)
hideConfirmation true:|![BG_AS_02_ComplexVMHandler_21](/images/2017-01/BG_AS_02_ComplexVMHandler_21.png)

Basics can contain elements of the following types:

![BG_AS_02_ComplexVMHandler_22](/images/2017-01/BG_AS_02_ComplexVMHandler_22.png)

##### Steps

Multiple steps are defined and each form a blade.

![BG_AS_02_ComplexVMHandler_23](/images/2017-01/BG_AS_02_ComplexVMHandler_23.png)

###### Resource Customization step

Right after the Basics, the Resource Customization step is defined.

```json
"name": "resourceCustomization",
"label": "Resource Customization",
"subLabel": {
    "preValidation": "Configure Resource Customization",
    "postValidation": "Done"
},
"bladeTitle": "Resource Customization",
"bladeSubtitle": "Configure Resource Customization",
"elements": [
    {
        "type": "Microsoft.Compute.SizeSelector",
        "name": "vmSize",
        "label": "Virtual Machine Size",
        "count": 1,
        "osPlatform": "Windows",
        "recommendedSizes": [
            "Standard_A2",
            "Standard_A3",
            "Standard_A4"
        ],
        "constraints": {
            "allowedSizes": [
                "Standard_A0",
                "Standard_A1",
                "Standard_A2",
                "Standard_A3",
                "Standard_A4"
            ]
        },
        "imageReference": {
            "offer": "WindowsServer",
            "publisher": "Microsoft",
            "sku": "2016-Standard-Core"
        }
    },
    {
        "type": "Microsoft.Common.TextBox",
        "constraints": {
            "regex": "^\\d+$",
            "validationMessage": "Value must be an integer.",
            "required": true
        },
        "defaultValue": "100",
        "name": "sqlDataDiskSize",
        "label": "Provide SQL Data Disk size in GB."
    },
    {
        "type": "Microsoft.Common.TextBox",
        "constraints": {
            "regex": "^\\d+$",
            "validationMessage": "Value must be an integer.",
            "required": true
        },
        "defaultValue": "50",
        "name": "sqlLogDiskSize",
        "label": "Provide SQL Log Disk size in GB."
    }
```

Which is rendered as:

![BG_AS_02_ComplexVMHandler_24](/images/2017-01/BG_AS_02_ComplexVMHandler_24.png)

The input parameters:

* vmSize: uses the Microsoft.Compute.SizeSelector type. It is constrained to a certain amount of allowed sized and has a couple of recommended sizes. The osPlatform, offer, publisher and sku properties are used for billing purposes (cost prediction) not (yet) available in Azure Stack.

Name|Result
----|------
Default size is first in recommended sizes:|![BG_AS_02_ComplexVMHandler_25](/images/2017-01/BG_AS_02_ComplexVMHandler_25.png)
The recommended sizes:|![BG_AS_02_ComplexVMHandler_26](/images/2017-01/BG_AS_02_ComplexVMHandler_26.png)
All allowed sizes:|![BG_AS_02_ComplexVMHandler_27](/images/2017-01/BG_AS_02_ComplexVMHandler_27.png)

* sqlDataDiskSize: uses the Microsoft.Common.Textbox type. Input is required and the value is constrained to a pattern match with a defined regex (numbers only). The input is used to specify the Data Disk size in GB.

Name|Result
----|------
Numbers allowed:|![BG_AS_02_ComplexVMHandler_28](/images/2017-01/BG_AS_02_ComplexVMHandler_28.png)
Letters not allowed:|![BG_AS_02_ComplexVMHandler_29](/images/2017-01/BG_AS_02_ComplexVMHandler_29.png)
Must contain value:|![BG_AS_02_ComplexVMHandler_30](/images/2017-01/BG_AS_02_ComplexVMHandler_30.png)

* sqlLogDiskSize: uses the Microsoft.Common.Textbox type. Input is required and the value is constrained to a pattern match with a defined regex (numbers only). The input is used to specify the Log Disk size in GB.

###### SQL Settings

The SQL Settings is the last defined step.

```json
"name": "sqlSettings",
"label": "SQL Settings",
"subLabel": {
    "preValidation": "Configure SQL Settings",
    "postValidation": "Done"
},
"bladeTitle": "SQL Settings",
"bladeSubtitle": "Configure SQL Settings",
"elements": [
    {
        "type": "Microsoft.Common.TextBox",
        "constraints": {
            "required": false
        },
        "label": "SQL Product key",
        "name": "sqlProductKey",
        "toolTip": "if you want to use the trial, leave empty.",
        "defaultValue": ""
    },
    {
        "type": "Microsoft.Common.DropDown",
        "name": "sqlFeatures",
        "label": "Select SQL Features",
        "constraints": {
            "allowedValues": [
                {
                    "label": "SQL Engine",
                    "value": "SQLENGINE"
                },
                {
                    "label": "Integration Services",
                    "value": "IS"
                },
                {
                    "label": "SQL Engine + IS",
                    "value": "SQLENGINE, IS"
                }
            ],
            "required": true
        }
    },
    {
        "type": "Microsoft.Common.Section",
        "name": "sqlEngine",
        "label": "SQL Engine",
        "visible": "[contains(steps('sqlSettings').sqlFeatures, 'SQLENGINE')]",
        "elements": [
            {
                "type": "Microsoft.Common.TextBox",
                "name": "sqlInstanceName",
                "label": "Instance name",
                "defaultValue": "MSSQLSERVER",
                "constraints": {
                    "required": true,
                    "regex": "(?!^[dD][eE][fF][aA][uU][lL][tT]$)(^[a-zA-Z_][0-9a-zA-Z$_]{1,15}$)",
                    "validationMessage": "Instance name must comply with Instance name rules."
                },
                "toolTip": "1-16 chars, can start with '_', cannot contain illigal characters (!, #, :, ;), cannot be named 'Default'"
            },
            {
                "type": "Microsoft.Common.TextBox",
                "constraints": {
                    "regex": "^\\d+$",
                    "validationMessage": "Value must be an integer."
                },
                "defaultValue": "1433",
                "name": "sqlInstancePort",
                "label": "Instance Port",
                "toolTip": "Default port is 1433"
            },
            {
                "type": "Microsoft.Common.OptionsGroup",
                "constraints": {
                    "allowedValues": [
                        {
                            "label": "SQL",
                            "value": "SQL"
                        },
                        {
                            "label": "Windows",
                            "value": "Windows"
                        }
                    ],
                    "required": true
                },
                "defaultValue": "SQL",
                "label": "Authentication Mode",
                "name": "sqlAuthMode",
                "toolTip": "When selecting Windows, you might not be able to remote connect SSMS until the VM is joined to a domain."
            }
        ]
    }
```

Which is rendered as:

![BG_AS_02_ComplexVMHandler_31](/images/2017-01/BG_AS_02_ComplexVMHandler_31.png)

This time, there is a context sensitive submenu which only appears when the SQL Features contain the SQLENGINE feature.

![BG_AS_02_ComplexVMHandler_32](/images/2017-01/BG_AS_02_ComplexVMHandler_32.png)

The SQL Engine settings are bound within a section called “SQL Engine”. You can see the label of the section appear above the elements that belong to the section. The section has the visible attribute defined which is linked to the sqlFeatures parameter outcome. **"[contains(steps(‘sqlSettings’).sqlFeatures, ‘SQLENGINE’)]"** defines if the sqlFeatures outcome contains the value **SQLENGINE**.

The input parameters:

* sqlProductKey: uses the Microsoft.Common.Textbox type. No input is required and the default value is an empty string. If the user wants to bring in their SQL license key, they can do so via this input or later on after installation is finished (you cannot downgrade a trail installation to standard as it defaults to enterprise feature set).
* sqlFeatures: uses the Microsoft.Common.DropDown type. There is no default set but a selection is required. The DropDown box has a set of predefined items from which the user can pick. The user interacts with the labels and the values are used internally.

Name|Result
----|------
Required:|![BG_AS_02_ComplexVMHandler_33](/images/2017-01/BG_AS_02_ComplexVMHandler_33.png)
Dropdown listing:|![BG_AS_02_ComplexVMHandler_34](/images/2017-01/BG_AS_02_ComplexVMHandler_34.png)
Selection:|![BG_AS_02_ComplexVMHandler_35](/images/2017-01/BG_AS_02_ComplexVMHandler_35.png)

* sqlInstanceName: uses the Microsoft.Common.Textbox type. Input is required and the value is constrained to a pattern match with a defined regex (valid Instance names only, https://technet.microsoft.com/en-us/library/ms143531(v=sql.90).aspx). The input is used to specify the SQL Instance Name. Defaults to MSSQLSERVER, which is also referred to as the default instance.

Name|Result
----|------
Valid instance name:|![BG_AS_02_ComplexVMHandler_36](/images/2017-01/BG_AS_02_ComplexVMHandler_36.png)
Tooltip hints:|![BG_AS_02_ComplexVMHandler_37](/images/2017-01/BG_AS_02_ComplexVMHandler_37.png)
Value required:|![BG_AS_02_ComplexVMHandler_38](/images/2017-01/BG_AS_02_ComplexVMHandler_38.png)
First letter not being underscore or a letter:|![BG_AS_02_ComplexVMHandler_39](/images/2017-01/BG_AS_02_ComplexVMHandler_39.png)
Over 16 characters:|![BG_AS_02_ComplexVMHandler_40](/images/2017-01/BG_AS_02_ComplexVMHandler_40.png)
Keyword:|![BG_AS_02_ComplexVMHandler_41](/images/2017-01/BG_AS_02_ComplexVMHandler_41.png)
Valid complex name:|![BG_AS_02_ComplexVMHandler_42](/images/2017-01/BG_AS_02_ComplexVMHandler_42.png)
Illegal characters:|![BG_AS_02_ComplexVMHandler_43](/images/2017-01/BG_AS_02_ComplexVMHandler_43.png)

* sqlInstancePort: uses the Microsoft.Common.Textbox type. Input is required and the value is constrained to a pattern match with a defined regex (numbers only). The input is used to specify the Port number on which the SQL instance listens. Besides the TCP listening port, there will also be a Windows firewall rule and NSG allow rule created for this port.
* sqlAuthMode: uses the Microsoft.Common.OptionsGroup type. A user selects the desired authentication scheme used by the SQL Server. SQL or Windows with default to SQL. If SQL is selected, a SQL login will be created using the provided administrator username and password.

Name|Result
----|------
SQL selected:|![BG_AS_02_ComplexVMHandler_44](/images/2017-01/BG_AS_02_ComplexVMHandler_44.png)
Windows selected:|![BG_AS_02_ComplexVMHandler_45](/images/2017-01/BG_AS_02_ComplexVMHandler_45.png)
Tooltip:|![BG_AS_02_ComplexVMHandler_46](/images/2017-01/BG_AS_02_ComplexVMHandler_46.png)

###### Outputs

Finally, the CreateUIDefinition contains an Outputs section.

```json
"adminUsername": "[basics('adminUsername')]",
"adminPassword": "[basics('adminPassword').password]",
"vmName": "[basics('vmName')]",
"vmSize": "[steps('resourceCustomization').vmSize]",
"sqlPort": "[int(steps('sqlSettings').sqlEngine.sqlInstancePort)]",
"sqlInstanceName": "[steps('sqlSettings').sqlEngine.sqlInstanceName]",
"sqlDataDiskSize": "[int(steps('resourceCustomization').sqlDataDiskSize)]",
"sqlLogDiskSize": "[int(steps('resourceCustomization').sqlLogDiskSize)]",
"sqlFeatures": "[steps('sqlSettings').sqlFeatures]",
"sqlAuthenticationMode": "[steps('sqlSettings').sqlEngine.sqlAuthMode]",
"sqlPID": "[steps('sqlSettings').sqlProductKey]"
```

This does not generate any UI but instead passed all user defined input to the deployment template. When you have developer-tools opened in your browser, you can see the outputs send to the console stream if you are using the CreateUIDefinition preview functionality:

![BG_AS_02_ComplexVMHandler_47](/images/2017-01/BG_AS_02_ComplexVMHandler_47.png)

If you have selected only to install Integration Services, you will notice all outputs belonging to the SQLENGINE section are not present. They are also missing from the summary blade:

![BG_AS_02_ComplexVMHandler_48](/images/2017-01/BG_AS_02_ComplexVMHandler_48.png)

##### Images

Now we have the hard part done, we need some pictures so the resulting gallery items will be quickly discoverable.

Name|Pixels|Image
----|------|-----
Wide|255×115|![BG_AS_02_ComplexVMHandler_49](/images/2017-01/BG_AS_02_ComplexVMHandler_49.png)
Large|115×115|![BG_AS_02_ComplexVMHandler_50](/images/2017-01/BG_AS_02_ComplexVMHandler_50.png)
Medium|90×90|![BG_AS_02_ComplexVMHandler_51](/images/2017-01/BG_AS_02_ComplexVMHandler_51.png)
Small|40×40|![BG_AS_02_ComplexVMHandler_52](/images/2017-01/BG_AS_02_ComplexVMHandler_52.png)
Screenshot0|533×324|![BG_AS_02_ComplexVMHandler_53](/images/2017-01/BG_AS_02_ComplexVMHandler_53.png)

#### UIDefinition

The UIDefinition.json is a bit of a mystery still. The schema uri doesn’t lead anywhere and there is no documentation (yet). The following UIDefinition works though and it should be saved in directly under the root folder. From the json we can see that it contains portal instructions on what blade type and extension to use.

```json
{
    "$schema": "https://gallery.azure.com/schemas/2015-02-12/UIDefinition.json#",
    "createDefinition": {
        "createBlade": {
            "name": "CreateMultiVmWizardBlade",
            "extension": "Microsoft_Azure_Compute"
        }
    }
}
```

#### Manifest

Finally, we need to reference everything in a manifest file.

```json
{
    "$schema": "https://gallery.azure.com/schemas/2015-10-01/manifest.json#",
    "name": "SQL2016SP1onServer2016Core",
    "publisher": "bgelens",
    "version": "1.0.0",
    "displayName": "SQL 2016 SP1 on Windows Server 2016 Core",
    "publisherDisplayName": "@bgelens",
    "publisherLegalName": "@bgelens",
    "summary": "Deploys a Windows Server 2016 Core Standard server with SQL Server 2016 SP1 installed.",
    "longSummary": "ms-resource:longSummary",
    "description": "ms-resource:description",
    "uiDefinition": {
        "path": "UIDefinition.json"
    },
    "artifacts": [
        {
            "name": "DefaultTemplate",
            "type": "Template",
            "path": "DeploymentTemplates\\mainTemplate.json",
            "isDefault": true
        },
        {
            "name": "CreateUIDefinition",
            "type": "Custom",
            "path": "DeploymentTemplates\\CreateUIDefinition.json",
            "isDefault": false
        }
    ],
    "categories": [
        "@bgelens"
    ],
    "links": [
        {
            "displayName": "SQL Gallery Item Git Repository.",
            "uri": "https://github.com/bgelens/MASComplexGalleryItem"
        }
    ],
    "images": [
        {
            "context": "ibiza",
            "items": [
                {
                    "id": "small",
                    "path": "Images\\Small.png",
                    "type": "icon"
                },
                {
                    "id": "medium",
                    "path": "Images\\Medium.png",
                    "type": "icon"
                },
                {
                    "id": "large",
                    "path": "Images\\Large.png",
                    "type": "icon"
                },
                {
                    "id": "wide",
                    "path": "Images\\Wide.png",
                    "type": "icon"
                },
                {
                    "id": "screenshot0",
                    "path": "Images\\Screenshot0.png",
                    "type": "screenshot"
                }
            ]
        }
    ]
}
```

It is possible to add localized data from resjon files into this manifest by using the ms-resource references. In this case I chose to ignore most of this possibility to avoid addition of unnecessary complexity. At a bare minimum, though, the AzureGalleryPackager checks for longSummery and description to come from this (default / fall-back) resjon file.

Before we can create a gallery item we need to add a strings folder to our gallery item and add a resources.resjon file containing the following:

```json
{
    "longSummary": "Deploys a Windows Server 2016 Core Standard server with SQL Server 2016 SP1 installed.",
    "description": "Deploys a Windows Server 2016 Core Standard server with SQL Server 2016 SP1 installed."
}
```

All metadata concerned with the gallery item is contained within the manifest. Also, the references to the images and deployment templates are relatively defined.

The gallery item will show up in its own category which is defined in the manifest as well. In this case, **"@bgelens"**.

> Warning:
> It is possible you created the manifest using a different schema. The AzureGalleryPackager has support for 3 sets of schema (2014-09-01, 2015-04-01 and 2015-10-01). I had to use ILSpy to peak into the AzureGalleryPackager dlls to figure this out. Depending on the schema, different keys and constructs are expected. In this case the latest version is used which expects the images construct for example. The older versions expect an “icons” construct instead. Also, the 2014 version doesn’t support the custom artefact type which is necessary for what we try to achieve here (in other words, the 2014 version is not compatible with the intend).

## Creating the azpkg

Now we edited the files, it’s time to create the azpkg we need to add the Gallery item to the portal. Download the [Azure Gallery Packager](http://aka.ms/azurestackmarketplaceitem){:target="_blank"}, unblock it and extract it somewhere you can find it. Let’s move the folder of our gallery item to the root of C. Now open a command prompt or PowerShell console and navigate to the packager and run: .\AzureGalleryPackager.exe -m C:\Sql2016SP1onServer2016Core\Manifest.json -o C:\MyMarketPlaceItems

Note you should create the target folder before as the packager will otherwise fail.

You should now have a marketplace item in azpkg format. If the packager did not work out, you probably have made some sort of typo somewhere or it just doesn’t like the paths you provide.

### Adding the Gallery Item to Azure Stack

Earlier on, a storage account was created which is going to be used again. If you did not create the storage account, please do so now by following the instructions from uploading the OS Image.

Run the following script to add the azpkg to Azure Stack (I assume you already setup the Azure Stack connection in PowerShell).

```powershell
$subscriptionid = (Get-AzureRmSubscription -SubscriptionName 'Default Provider Subscription').SubscriptionId
$StorageAccount = Get-AzureRmStorageAccount -ResourceGroupName tenantartifacts -Name tenantartifacts
$GalleryContainer = $StorageAccount | New-AzureStorageContainer -Name gallery -Permission Blob
$SQL2016Azpkg = $GalleryContainer | Set-AzureStorageBlobContent -File C:\MyMarketPlaceItems\bgelens.SQL2016SP1onServer2016Core.1.0.0.azpkg
Add-AzureRMGalleryItem -SubscriptionId $subscriptionid -GalleryItemUri $SQL2016Azpkg.ICloudBlob.Uri -Apiversion "2015-04-01" -Verbose
```

You should see a message with StatusCode Ok (if you get created, just run it again to be sure).

![BG_AS_02_ComplexVMHandler_54](/images/2017-01/BG_AS_02_ComplexVMHandler_54.png)

Now we login to the portal, wait a bit, refresh a couple of times and eventually we get:

![BG_AS_02_ComplexVMHandler_55](/images/2017-01/BG_AS_02_ComplexVMHandler_55.png)

### Deploying the Gallery Item

Now we have added the Gallery Item we can deploy it.

![BG_AS_02_ComplexVMHandler_56](/images/2017-01/BG_AS_02_ComplexVMHandler_56.png)

We fill out the Basics with the Administrator user and password plus the VM name. Also, we create a new resource group with this deployment. At the Resource Customization step we define we want a 200GB SQL Data disk and a 100GB Log disk.

![BG_AS_02_ComplexVMHandler_57](/images/2017-01/BG_AS_02_ComplexVMHandler_57.png)

Next, at the SQL Settings step, we define we want both the SQL Engine and IS, name the instance and provide a non-default port, 10001. Finally, we hit the summer where the portal validates the input. The tenant user can download the deployment template for quick redeployment.

![BG_AS_02_ComplexVMHandler_58](/images/2017-01/BG_AS_02_ComplexVMHandler_58.png)

The portal adds a Buy step where the tenant user agrees with the purchase. At this time, we cannot disable this behavior. After the purchase is done, deployment is started.

![BG_AS_02_ComplexVMHandler_59](/images/2017-01/BG_AS_02_ComplexVMHandler_59.png)

The tenant user can follow the deployment as it is happening. In this screenshot, we can see that currently the VM is being deployed and all prerequisite steps where successful.

![BG_AS_02_ComplexVMHandler_60](/images/2017-01/BG_AS_02_ComplexVMHandler_60.png)

Now the deployment is finished, we can validate that we get what we expected.

![BG_AS_02_ComplexVMHandler_61](/images/2017-01/BG_AS_02_ComplexVMHandler_61.png)

First let’s look at the NSG to see if the custom SQL port has been opened.

![BG_AS_02_ComplexVMHandler_62](/images/2017-01/BG_AS_02_ComplexVMHandler_62.png)

As you can see, the 10001 port is opened. Let’s check the DSC extension status.

![BG_AS_02_ComplexVMHandler_63](/images/2017-01/BG_AS_02_ComplexVMHandler_63.png)

Finally, let’s look up the public IP address and open a connection with the deployed SQL instance.

![BG_AS_02_ComplexVMHandler_64](/images/2017-01/BG_AS_02_ComplexVMHandler_64.png)

The IP address is 192.168.102.27.

![BG_AS_02_ComplexVMHandler_65](/images/2017-01/BG_AS_02_ComplexVMHandler_65.png)

![BG_AS_02_ComplexVMHandler_66](/images/2017-01/BG_AS_02_ComplexVMHandler_66.png)

#### Testing in production

Developing and iterating on the gallery items is probably something you do not want to do in public. Having a secondary Azure Stack available for this development can be costly so luckily we can easily develop in production without tenants seeing what we do.

First, as we already explained before, you can preview the CreateUIDefinition by uploading it to a public location and have the portal pointed towards it.

Next, it is also possible to add a filter to the Gallery Item manifest which allows you to hide the Gallery Item from general sight.

```json
"filters": [
    {
        "type": "HideKey",
        "value": "MyNewGI"
    }
]
```

6
"filters": [
    {
        "type": "HideKey",
        "value": "MyNewGI"
    }
]
When you use this approach, you can find the Gallery Item by appending a query key to the azurestack uri. E.g: https://portal.azurestack.local?microsoft_azure_marketplace_ItemHideKey=MyNewGI

### Resources

I found a GitHub repository explaining more of the available UI elements and how they are rendered. If what you need is not in here and not obvious through the schema, take a look here. Also, the Gallery Item GitHub repository for Elasticsearch has been a great resource as well as the Azure Portal documentation.

### Conclusion

That’s it! I hope this helps you with creating your own Gallery Items. This was a pretty simple one, but imagine this example to be extended with a number of VMs parameter making use of the [copy functionality](https://docs.microsoft.com/en-us/azure/azure-resource-manager/resource-group-create-multiple){:target="_blank"} and having settable options like, enable contained database support and others. You could use this as an easy way to deploy your SQL Resource Provider SQL hosts with.

You could also make more complex multi-tier application Gallery Items like SQL AlwaysOn or a RDS farm for example. Finally, we won’t be held back anymore. Gone are the days with the VM Role limitation of Azure Pack!

Remember, this has been tested on TP2 and because of that, in no means I can guarantee it will work on TP3 and on. But if this happens, be sure I’ll start my journey all over again :)

You can find the source files I used, together with the DSC resources, scripts and azpkg on my GitHub repo [here](https://github.com/bgelens/MASComplexGalleryItem){:target="_blank"}!
