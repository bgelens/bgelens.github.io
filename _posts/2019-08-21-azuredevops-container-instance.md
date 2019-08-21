---
title:  "Azure ARM Linked Templates and Complete mode"
date:   2019-08-21 12:00:00
categories: ["Azure","ARM","AzureDevOps","Docker","PowerShell","ACI"]
tags: ["Azure","ARM","AzureDevOps","Docker","PowerShell","ACI"]
excerpt_separator: <!--more-->
---

The Microsoft provided hosted build agents for Azure DevOps might not suite all requirements. E.g. the Az PowerShell modules on the images provided by Microsoft are lagging behind. To compensate, in general pipelines spend a lot of time installing dependencies to complete the job at hand. Having a custom build agent can resolves these issues as the dependencies are installed at image creation and available from there on, thus these beter suite your needs (build for purpose).

You can host build agents on any compute platform. For this solution we build docker images and host them on Azure Container Instances as it is relatively easy to create container images containing all the requirements compared to VMs. It is also far easier creating new versions as the creation of the image is fully automated.

<!--more-->

## Create Agent Pool

To create an Agent Pool please follow the [documentation](https://docs.microsoft.com/en-us/azure/devops/pipelines/agents/pools-queues?view=azure-devops#creating-agent-pools).

## Build container image

For this blog we'll create images for both Windows and Linux.

### Linux

The image run commands are based on the [Linux Azure DevOps agent guide](https://docs.microsoft.com/en-us/azure/devops/pipelines/agents/v2-linux?view=azure-devops).

The Linux image is based on `mcr.microsoft.com/powershell:6.2.0-ubuntu-18.04` which is an Ubuntu 18.04 image with PowerShell Core 6.2 pre-installed managed by Microsoft (see [DockerHub](https://hub.docker.com/_/microsoft-powershell) for more info).

Safe the following Dockerfile as `Linux.Dockerfile`:

```docker
# escape=`

FROM mcr.microsoft.com/powershell:6.2.0-ubuntu-18.04

ARG agentversion=2.155.1
ENV agentversion=${agentversion}

SHELL [ "pwsh", "-NoProfile", "-Command" ]

ENV AGENT_ALLOW_RUNASROOT 1

WORKDIR /agent

RUN $ProgressPreference = 'SilentlyContinue' ; `
    Invoke-WebRequest -Uri "https://vstsagentpackage.azureedge.net/agent/$env:agentversion/vsts-agent-linux-x64-$env:agentversion.tar.gz" -OutFile "./vsts-agent-linux-x64-$env:agentversion.tar.gz" -UseBasicParsing ; `
    tar  zxvf "./vsts-agent-linux-x64-$env:agentversion.tar.gz" ; `
    rm "./vsts-agent-linux-x64-$env:agentversion.tar.gz" ; `
    ./bin/installdependencies.sh ; `
    Install-Module Az -Force -Scope AllUsers ; `
    Install-Module Pester -Force -Scope AllUsers ; `
    Install-Module PSScriptAnalyzer -Force -Scope AllUsers ; `
    apt install git -y ; `
    Invoke-WebRequest -Uri https://packages.microsoft.com/config/ubuntu/18.04/packages-microsoft-prod.deb -OutFile ./packages-microsoft-prod.deb -UseBasicParsing ; `
    dpkg -i ./packages-microsoft-prod.deb ; `
    rm ./packages-microsoft-prod.deb ; `
    apt-get install apt-transport-https -y ; `
    apt-get update ; `
    apt-get install dotnet-sdk-2.2 -y ; `
    apt-get install zip -y

ENTRYPOINT [ "/bin/bash", "-c", "./config.sh --unattended --replace && ./run.sh" ]
```

As you can see, I preselected some things to install like the Az modules, Pester and PSScriptAnalyzer. Now to build the image:

```sh
docker build --force-rm -t azd-ubuntu1804-pwsh6.2.0 -f Linux.Dockerfile .
```

> Note that the Agent Version downloaded is passed in via arguments with a default of 2.155.1 which is the latest at the time of writing. You can overwrite by adding the `--build-arg` argument to `docker build`. E.g: `docker build --force-rm -t azd-ubuntu1804-pwsh6.2.0 --build-arg agentversion=2.150.3 -f ./Linux.Dockerfile .` to make use of agent version `2.150.3`.

### Windows

The Windows image is based on `mcr.microsoft.com/powershell:6.2.0-windowsservercore-1809` which is a Windows Server 2019 / 1809 image with PowerShell Core 6.2 pre-installed managed by Microsoft (see [DockerHub](https://hub.docker.com/_/microsoft-powershell) for more info).

Safe the following Dockerfile as `Windows.Dockerfile`:

```docker
# escape=`

FROM mcr.microsoft.com/powershell:6.2.0-windowsservercore-1809

ARG agentversion=2.155.1
ENV agentversion=${agentversion}

SHELL [ "pwsh", "-NoProfile", "-Command" ]

WORKDIR /agent

COPY [ "DnsFix.ps1", "./DnsFix.ps1" ]

RUN $ProgressPreference = 'SilentlyContinue' ; `
    Invoke-WebRequest -Uri "https://vstsagentpackage.azureedge.net/agent/$env:agentversion/vsts-agent-win-x64-$env:agentversion.zip" -OutFile "./vsts-agent-win-x64-$env:agentversion.zip" -UseBasicParsing ; `
    Expand-Archive -Path "./vsts-agent-win-x64-$env:agentversion.zip" -DestinationPath . ; `
    Remove-Item "./vsts-agent-win-x64-$env:agentversion.zip" ; `
    New-Item -ItemType Directory -Name Modules -Path c:\ -Force ; `
    Save-Module Az -Path c:\Modules ; `
    Save-Module Pester -Path c:\Modules ; `
    Save-Module PSScriptAnalyzer -Path c:\Modules ; `
    Invoke-WebRequest -Uri https://download.visualstudio.microsoft.com/download/pr/3c43f486-2799-4454-851c-fa7a9fb73633/673099a9fe6f1cac62dd68da37ddbc1a/dotnet-sdk-2.2.203-win-x64.exe -OutFile ./dotnet-sdk-2.2.203-win-x64.exe ; `
    Start-Process ./dotnet-sdk-2.2.203-win-x64.exe -ArgumentList '-q' -Wait ; `
    Remove-Item ./dotnet-sdk-2.2.203-win-x64.exe ; `
    Invoke-WebRequest -Uri https://github.com/git-for-windows/git/releases/download/v2.21.0.windows.1/Git-2.21.0-64-bit.exe -OutFile ./Git-2.21.0-64-bit.exe ; `
    Start-Process ./Git-2.21.0-64-bit.exe -ArgumentList ' /VERYSILENT /NORESTART /SUPPRESSMSGBOXES /NOCANCEL /SP-' -Wait ; `
    Remove-Item ./Git-2.21.0-64-bit.exe ; `
    $machinePath = [environment]::GetEnvironmentVariable('path', [System.EnvironmentVariableTarget]::Machine) ; `
    $newMachinePath = 'C:\Program Files\Git\mingw64\bin;C:\Program Files\Git\usr\bin;C:\Program Files\Git\bin;' + $machinePath ; `
    [environment]::SetEnvironmentVariable('path', $newMachinePath, [System.EnvironmentVariableTarget]::Machine) ; `
    $config = cat $PSHOME/powershell.config.json | ConvertFrom-Json -AsHashtable ; `
    $config.Add('PSModulePath','%ProgramFiles%\PowerShell\Modules;%ProgramFiles%\powershell\latest\Modules;%windir%\system32\WindowsPowerShell\v1.0\Modules;C:\Modules') ; `
    $config | ConvertTo-Json | Out-File $PSHOME/powershell.config.json -Force ; `
    $machineV5ModPath = [environment]::GetEnvironmentVariable('PSModulePath', [System.EnvironmentVariableTarget]::Machine) ; `
    $newMachineV5ModPath = 'c:\Modules;' + $machineV5ModPath ; `
    [environment]::SetEnvironmentVariable('PSModulePath', $newMachineV5ModPath, [System.EnvironmentVariableTarget]::Machine)

ENTRYPOINT [ "C:\\Windows\\system32\\cmd.exe", "/C", "pwsh -f .\\DnsFix.ps1 -nol -noni -nop && .\\config.cmd --unattended --replace && .\\run.cmd" ]
```

Again, as with the Linux image, I preselected some things to install like the Az modules, Pester and PSScriptAnalyzer. There is also a `DnsFix.ps1` that is copied in the image to [workaround an issue with Azure Container Instances and Windows containers](https://stackoverflow.com/questions/55933295/unable-to-do-dns-lookup-in-azure-container-instance-windows-container){:target="_blank"}. I've hit this issue myself hence the inclusion of this fix (have the container make use of google dns servers). Make sure you save the following as `DnsFix.ps1` in the same folder as you saved the dockerfile.

```powershell
# this is a workaround for dns issues with Windows Containers in ACI
$dnsClientSettings = Get-DnsClientServerAddress -AddressFamily IPv4 | Where-Object -FilterScript { $_.ElementName -NotLike "Loopback*" }
$dnsClientSettings | Set-DnsClientServerAddress -ServerAddresses @('8.8.8.8', '8.8.4.4')
```

Now to build the image:

```sh
docker build --force-rm -t azd-windows1809-pwsh6.2.0 -f Windows.Dockerfile .
```

## Running locally

The image requires a PAT token to connect with Azure DevOps. See the [documentation](https://docs.microsoft.com/en-us/azure/devops/organizations/accounts/use-personal-access-tokens-to-authenticate?view=azure-devops) on how to acquire one.

To test the resulting image locally run:

```sh
PATTOKEN="get a pat token from azure devops"
docker run --rm -it -e VSTS_AGENT_INPUT_URL=https://dev.azure.com/<MyOrchName> -e VSTS_AGENT_INPUT_AUTH=pat -e VSTS_AGENT_INPUT_TOKEN=$PATTOKEN -e VSTS_AGENT_INPUT_POOL=<MyPoolName> -e VSTS_AGENT_INPUT_AGENT=agent-0 --user azd:azd azd-ubuntu1804-pwsh6.2.0
```

## Publish container image to ACR

This blog assumes an Azure Container Registry is already created. See the [guides](https://docs.microsoft.com/en-us/azure/container-registry/container-registry-get-started-portal) on how to handle this task if not already created.

To publish the just created image to Azure Container Registry, make sure that the registry is enabled for [admin access](https://docs.microsoft.com/en-us/azure/container-registry/container-registry-authentication#admin-account) so you have a username and password to authenticate to it.

Login and provide the username and password:

```sh
docker login <myregistry>.azurecr.io
```

Tag the image so docker knows where to send it:

```sh
# linux
docker tag azd-ubuntu1804-pwsh6.2.0 <myregistry>.azurecr.io/azd/azd-ubuntu1804-pwsh6.2.0

# windows
docker tag azd-windows1809-pwsh6.2.0 <myregistry>.azurecr.io/azd/azd-windows1809-pwsh6.2.0
```

Push the image:

```sh
# linux
docker push <myregistry>.azurecr.io/azd/azd-ubuntu1804-pwsh6.2.0

# windows
docker push <myregistry>.azurecr.io/azd/azd-windows1809-pwsh6.2.0
```

## Deploy container instance

To deploy / update an Azure Container Instance, I've prepared and ARM template.

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2015-01-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "containerInstanceName": {
      "type": "string",
      "metadata": {
        "description": "Specifies the name of the Container Instance"
      }
    },
    "containerCount": {
      "type": "int",
      "minValue": 1,
      "defaultValue": 1,
      "metadata": {
        "description": "Specifies the amount of containers to deploy to the Container Instance. (for Windows the maximum is 1. You cannot update the count after deployment)"
      }
    },
    "patToken": {
      "type": "securestring",
      "metadata": {
        "description": "The personal access token used by the agent(s) to connect to Azure DevOps"
      }
    },
    "azdUrl": {
      "type": "string",
      "metadata": {
        "description": "The url of the Azure DevOps project (https://dev.azure.com/<projectName>)"
      }
    },
    "agentPool": {
      "type": "string",
      "metadata": {
        "description": "The pool in which the agent(s) should join"
      }
    },
    "agentNamePrefix": {
      "type": "string",
      "metadata": {
        "description": "The name prefix for the agent(s) (e.g. aci-agent-ubuntu- will result in aci-agent-ubuntu-1 for the first container and so on)"
      }
    },
    "registryLoginServer": {
      "type": "string",
      "metadata": {
        "description": "The Container Registry login url (e.g. <name>.azurecr.io) where the image will be pulled from"
      }
    },
    "registryUserName": {
      "type": "string",
      "metadata": {
        "description": "The Container Registry UserName"
      }
    },
    "registryPassword": {
      "type": "securestring",
      "metadata": {
        "description": "The Container Registry Password"
      }
    },
    "registryImageUri": {
      "type": "string",
      "metadata": {
        "description": "The Container Image Uri (e.g. <registryName>.azurecr.io/<libraryName>/<imageName>:<imageTag>)"
      }
    },
    "memoryInGb": {
      "type": "string",
      "defaultValue": "3.5",
      "metadata": {
        "description": "The amount of memory to assign to the container(s)"
      }
    },
    "cpuCount": {
      "type": "string",
      "defaultValue": "2",
      "metadata": {
        "description": "The amount of CPU(s) to assign to the container(s)"
      }
    },
    "osType": {
      "type": "string",
      "defaultValue": "Linux",
      "metadata": {
        "description": "The OSType of the container(s) correlating to the image being pulled"
      }
    },
    "restartPolicy": {
      "type": "string",
      "defaultValue": "Always",
      "metadata": {
        "description": "The restart policy for a container (Always, OnFailure, Never)"
      }
    },
    "assignManagedIdentity": {
      "type": "bool",
      "defaultValue": false,
      "metadata": {
        "description": "Specify if a Managed Identity should be assigned"
      }
    }
  },
  "variables": {},
  "resources": [
    {
      "type": "Microsoft.ContainerInstance/containerGroups",
      "apiVersion": "2018-10-01",
      "name": "[parameters('containerInstanceName')]",
      "location": "[resourceGroup().location]",
      "identity": "[if(parameters('assignManagedIdentity'), json('{\"type\": \"SystemAssigned\"}'), json('null'))]",
      "properties": {
        "copy": [
          {
            "name": "containers",
            "count": "[if(equals(parameters('osType'), 'Windows'), 1, parameters('containerCount'))]",
            "input": {
              "name": "[concat(parameters('agentNamePrefix'), padLeft(copyIndex('containers', 1), 3, '0'))]",
              "properties": {
                "image": "[parameters('registryImageUri')]",
                "environmentVariables": [
                  {
                    "name": "VSTS_AGENT_INPUT_URL",
                    "value": "[parameters('azdUrl')]"
                  },
                  {
                    "name": "VSTS_AGENT_INPUT_AUTH",
                    "value": "pat"
                  },
                  {
                    "name": "VSTS_AGENT_INPUT_TOKEN",
                    "secureValue": "[parameters('patToken')]"
                  },
                  {
                    "name": "VSTS_AGENT_INPUT_POOL",
                    "value": "[parameters('agentPool')]"
                  },
                  {
                    "name": "VSTS_AGENT_INPUT_AGENT",
                    "value": "[concat(parameters('agentNamePrefix'), padLeft(copyIndex('containers', 1), 3, '0'))]"
                  }
                ],
                "resources": {
                  "requests": {
                    "memoryInGB": "[parameters('memoryInGb')]",
                    "cpu": "[parameters('cpuCount')]"
                  }
                }
              }
            }
          }
        ],
        "imageRegistryCredentials": [
          {
            "server": "[parameters('registryLoginServer')]",
            "username": "[parameters('registryUserName')]",
            "password": "[parameters('registryPassword')]"
          }
        ],
        "restartPolicy": "[parameters('restartPolicy')]",
        "osType": "[parameters('osType')]"
      }
    }
  ],
  "outputs": {
    "principalId": {
      "condition": "[parameters('assignManagedIdentity')]",
      "type": "string",
      "value": "[reference(concat(resourceId('Microsoft.ContainerInstance/containerGroups/', parameters('containerInstanceName')), '/providers/Microsoft.ManagedIdentity/Identities/default'), '2015-08-31-preview').principalId]"
    }
  }
}
```

I've created some param files to go with the template:

Linux:

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2015-01-01/deploymentParameters.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "containerInstanceName": {
      "value": "<instanceName>"
    },
    "containerCount": {
      "value": 1
    },
    "azdUrl": {
      "value": "https://dev.azure.com/<MyOrg>"
    },
    "agentPool": {
      "value": "<myagentpool>"
    },
    "agentNamePrefix": {
      "value": "aci-agent-ubuntu-"
    },
    "registryLoginServer": {
      "value": "<myregistry>.azurecr.io"
    },
    "registryImageUri": {
      "value": "<myregistry>.azurecr.io/azd/azd-ubuntu1804-pwsh6.2.0:latest"
    }
  }
}
```

Windows:

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2015-01-01/deploymentParameters.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "containerInstanceName": {
      "value": "<instanceName>"
    },
    "containerCount": {
      "value": 1
    },
    "azdUrl": {
      "value": "https://dev.azure.com/<MyOrg>"
    },
    "agentPool": {
      "value": "<myagentpool>"
    },
    "agentNamePrefix": {
      "value": "aci-agent-windows-"
    },
    "registryLoginServer": {
      "value": "<myregistry>.azurecr.io"
    },
    "registryImageUri": {
      "value": "<myregistry>.azurecr.io/azd/azd-windows1809-pwsh6.2.0:latest"
    },
    "osType": {
      "value": "Windows"
    }
  }
}
```

Replace all values that have `<>` with your own.

Some arguments have to be provided securely. Deploy the template using PowerShell:

```powershell
# get the pat token from the user and store as secureString
$patToken = Read-Host -AsSecureString -Prompt patToken

# get registry credentials
$registry = Get-AzContainerRegistry -Name <myregistry> -ResourceGroupName <resourcegroup>
$registryCred = $registry | Get-AzContainerRegistryCredential
$secureRegistryPassword = ConvertTo-SecureString -String $registryCred.Password -AsPlainText -Force

# deploy container instance
New-AzResourceGroupDeployment -ResourceGroupName <resourcegroup> -TemplateFile ./mytemplate.json -TemplateParameterFile ./azd-windows-containerinstance.parameters.json -registryUserName $registryCred.Username -registryPassword $secureRegistryPassword -patToken $patToken
```

## ACI caveats

Some caveats I've found when working with ACI:

* Note you can enable Managed Identity by setting the `assignManagedIdentity` parameter to `true`. The template will output the principalId of the resulting identity so you can assign it a Role on a certain scope (subscription / resource group). Note that [Managed Identity for ACI](https://docs.microsoft.com/en-us/azure/container-instances/container-instances-managed-identity) is in preview at this time.
* Preliminary tests with Managed Identity did not work. The metadata endpoint http://169.254.169.254 was not available from within the container
* Managed Identity is not available at this time on Windows Container Instances.
* Windows Container Groups can only contain 1 container. Old [issue](https://github.com/MicrosoftFeedback/aci-issues/issues/10) but still the case. The template enforces a count of 1 in case of Windows instead of throwing an error. Once this limitation is lifted, the template if condition can be removed.
* Increasing the amount of containers post deployment by redeploying the template does not work. You need to delete / create a new Container Instance to do this.

## Using the new Agents

To use the new Agents, select the correct Agent Pool in the build / release / pipeline (the configuration is part of the Agent Job). Add a demand for `Agent.OS` with either `Windows_NT` or `Linux` if both agent types exist in the same pool.
