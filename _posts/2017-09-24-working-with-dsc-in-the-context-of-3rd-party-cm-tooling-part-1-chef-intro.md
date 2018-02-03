---
title:  "Working with DSC in the context of 3rd party CM tooling. Part 1: Chef intro"
date:   2017-09-24 11:30:00
categories: ["Desired State Configuration","Chef"]
tags: ["Desired State Configuration","Chef"]
---

Working with a DSC only implementation can be a bit challaging. In almost all of my experiences we ended up building some kind of custom solution around DSC to make it more usable, flexible, manageable and insightful. In this blog series I'll investigate some configuration management solutions to see if they can positively extend my experience working with DSC without the need to build a custom solution.

I'm planning to check out the following CM tools as part of this series:

* Chef
* Puppet
* Ansible

Index:

* [Part 1: Chef intro](http://bgelens.nl/working-with-dsc-in-the-context-of-3rd-party-cm-tooling-part-1-chef-intro)
* [Part 2: Chef notifications](http://bgelens.nl/working-with-dsc-in-the-context-of-3rd-party-cm-tooling-part-2-chef-notifications/)
* [Part 3: Chef server](http://bgelens.nl/working-with-dsc-in-the-context-of-3rd-party-cm-tooling-part-3-chef-server/)
* [Part 4: Chef bootstrapping](http://bgelens.nl/working-with-dsc-in-the-context-of-3rd-party-cm-tooling-part-4-chef-bootstrap/)
* [Part 5: Chef client settings](http://bgelens.nl/working-with-dsc-in-the-context-of-3rd-party-cm-tooling-part-5-chef-clientsettings/)
* [Part 6: Puppet intro](http://bgelens.nl/working-with-dsc-in-the-context-of-3rd-party-cm-tooling-part-6-puppet-intro/)
* [Part 7: Puppet notifications](http://bgelens.nl/working-with-dsc-in-the-context-of-3rd-party-cm-tooling-part-7-puppnotificationset/)
* [Part 8: Puppet server](http://bgelens.nl/working-with-dsc-in-the-context-of-3rd-party-cm-tooling-part-8-puppet-server/)

I'll work from DSC perspective as that is what I know. (e.g. Pull server model, authoring / staging / enact stage, etc and see where I'll end up). If you are more familiar than me with the CM tooling discussed, please excuse me if I'm understating or misdiscribing the functionality while comparing with DSC (I just don't know any better yet). The goal is to see where and how any of these tools (can) extent DSC or make use of DSC to be more flexible, capable, robust, etc.

I'm pretty much a total newbie on any of these tools (apart from some general understanding / playing around with) so I'm hoping this series will be helpful to me but also to anyone curious to what is possible.

As my experimentation nodes I'll be using Azure VMs.

So let's get started with the first post on using Chef in combination with DSC.

# Chef

![chef_logo](/images/2017-09/chef_logo.png){: height="200px"}

At the heart of Chef is the [chef client](https://docs.chef.io/chef_client_overview.html){:target="_blank"} which is the engine that does all the heavy lifting. You can compare it a bit with the Local Configuration Manager when talking about DSC. Normally it will operate in conjunction with a [chef server](https://docs.chef.io/server_components.html){:target="_blank"} implementation which you can view as the DSC Pull Server or Azure Automation DSC. From first look the chef server is a nice and rich manageable solution. There is also the hosted chef solution which is a PaaS offering with most of the functionality of chef server without the hassle of maintaining infrastructure and such.

When comparing with DSC we have found relations with the staging (chef server) and enacting / reporting (chef client) phase. What about the authoring? Authoring, as with DSC, is the process of manipulating text files to declare the end state of a node. In the case of chef it's done in ruby. Chef provides what is known as the ChefDK which is a toolset to help you out with authoring, staging, bootstrapping (have a chef-client installed on a remote node and have it register with a chef server in one go) and so on. At the hard of the ChefDK is a tool called knife which will become familiar to you very quickly.

# Deploy Azure VM using Azure CLI 2.0

After some investigation I found out that it is possible to work with chef-client in a local mode (for testing purposes) so there won't be a need to setup chef server / hosted chef immediately. Let's deploy a Windows Server 2016 Core machine to Azure using the Azure CLI 2.0 (I haven't played with it before but it's nice to explore right!). For instructions on how to install it see [https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest){:target="_blank"} (I'm running it in WSL). For instruction on how to get started with it see [https://docs.microsoft.com/en-us/cli/azure/get-started-with-azure-cli?view=azure-cli-latest](https://docs.microsoft.com/en-us/cli/azure/get-started-with-azure-cli?view=azure-cli-latest){:target="_blank"}

First login using ```az login``` and select the subscription you want to work with if you have more as one using ```az account set --subscription subscriptionid/name```

Create a resource group:

```sh
az group create -l westeurope -n chefdsc
```

I wanted to deploy a Server core image so I first had to find the urn to pass to the ```--image``` parameter of ```az vm create```. I was able to find it using:

```sh
az vm image list --output table --all --publisher Microsoft --sku core
```

Deploy a VM to the resource group:

```sh
az vm create -n chefdsc01 -g chefdsc --image MicrosoftWindowsServer:WindowsServer:2016-Datacenter-Server-Core:2016.127.20170811 --no-wait --size Standard_DS1_V2 --admin-username ben --admin-password 'My$uperSecretPa$$w0rd'
```

This will create a VM with a managed disk for the OS. It will also create a virtual network, public ip address, network security group. The nice thing about Azure CLI 2.0 is that it will deploy the resources using ARM templates instead of making API calls itself. This ensures that the deployment history is available to us by checking the resource group deployments.

Note the ```--no-wait``` switch! Hurray for async!

To see the deployment using Azure CLI in a concise output (since there is now only 1 deployment, this would be the one. If you have multiple, you need to use az group deployment show specifying the name of the deployment)

```sh
az group deployment list --resource-group chefdsc --output table
```

To see the deployment template itself in nice colored json output:

```sh
az group deployment list --resource-group chefdsc --output jsonc
```

You could choose to wait for the deployment to finish:

```sh
az group deployment wait --name vm_deploy_vUPh7TQkqric8q7OUilNJyDBsz0gLEq2 --resource-group chefdsc --created
```

Note, there is a VM Extension for Chef available. We'll look at that later.

The VM is provisioned with a public IP address so let's connect with it over RDP for now.
Let's find the public ip address first:

```sh
az vm list-ip-addresses -n chefdsc01 --output table
```

Now we have the ip address we can use mstsc to connect to it.

# Chef client and vs code

Now we are connected with the vm created earlier. Let's first install the chef-client (find the latest msi here:[https://downloads.chef.io/chef](https://downloads.chef.io/chef){:target="_blank"}). Start PowerShell and download the msi by running:

```powershell
Invoke-WebRequest -UseBasicParsing -Uri https://packages.chef.io/files/stable/chef/13.4.24/windows/2016/chef-client-13.4.24-1-x64.msi -OutFile c:\chef-client.msi
Unblock-File -Path C:\chef-client.msi
C:\chef-client.msi
```

![chef_setup01](/images/2017-09/chef_setup1.png){: width="600"}

![chef_setup01](/images/2017-09/chef_setup2.png){: width="600px"}

Installation takes a while, be patient.

Now that chef-client is installed. Let's install vs-code so we have a nice editor available.

```powershell
Invoke-WebRequest -UseBasicParsing -Uri https://go.microsoft.com/fwlink/?Linkid=852157 -OutFile c:\vscode.exe
Unblock-File -Path c:\vscode.exe
C:\vscode.exe
```

Make sure you select the option that code would be added to the Path. Logoff and log back in again.

I'll install the PowerShell extension and the chef extension (not sure yet if it will be beneficial for me)

```powershell
code --install-extension ms-vscode.PowerShell
code --install-extension Pendrica.Chef
```

# Start small

After reading in a bit on chef, I have found that the smallest configuration artifact is what they call a [recipe](https://docs.chef.io/recipes.html){:target="_blank"}. You can look at a recipe as if it were a PowerShell DSC configuration script.

Let's create a recipe. First let's create a folder c:\recipe (```mkdir c:\recipe```)  and open it with code (```code c:\chef```).

Now that vs code is opened. Let's create a file called dsc01.rb (rb is for ruby)
Start typing dsc and you'll see an auto complete snippet for dsc_resource. Hit enter.

```ruby
dsc_resource 'name' do
  resource :resource_name
  property :property_name, 'property_value'
end
```

The chef extension for code is helpful already! As stated earlier, I'm looking to see where chef is extending / improving my DSC experience so I'm looking for DSC specifics first. I'm sure we'll see some chef native resource usage as well.

So chef is able to consume DSC resources as if they were chef resource by making use of this [dsc_resource resource](https://docs.chef.io/resource_dsc_resource.html){:target="_blank"}. Basically this resource will call into the ```Test``` and ```Set``` methods of the resource without the need to compile a MOF and put in a DSC configuration. Actually when I started looking into the chef code base on GitHub I found that the ```dsc_resource resource``` is actually calling into ```Invoke-DscResource``` [https://github.com/chef/chef/blob/master/lib/chef/provider/dsc_resource.rb#L169](https://github.com/chef/chef/blob/master/lib/chef/provider/dsc_resource.rb#L169){:target="_blank"}. This makes the use of this resource dependent on WMF5+

Alternatively we could make use of the [dsc_script resource](https://docs.chef.io/resource_dsc_script.html){:target="_blank"} which takes the PowerShell configuration syntax and will compile a MOF and send it to the LCM for enacting (this is WMF4 compatible).

From the first look, making use of the ```dsc_resource``` resource would be preferred as it fully integrates DSC resources into the chef recipes.

Let's look at the syntax of ```dsc_resource``` resource when targeting the DSC inbox file resource.

```ruby
dsc_resource 'some file' do
    resource :File
    module_name 'PSDesiredStateConfiguration'
    module_version '1.1'
    property :DestinationPath, 'c:\testfile.txt'
    property :Contents, 'some text'
    property :Ensure, 'Present'
end
```

So we specify we want to use the ```dsc_resource``` resource and it has to have a name (from some testing I can say that unlike a PowerShell configuration, this name doesn't have to be unique per se). The ```do``` and ```end``` keywords are the curly brace ruby alternative. The resource has a lot of properties we can set

* Resource is mandatory
* Module_name and version are optional
* The DSC resource properties are mapped via property keys.

Let's apply this recipe.

```powershell
chef-client --local-mode .\dsc01.rb
```

![dscresources01](/images/2017-09/dscresres01.png){: width="600"}

As you can see the test method of the file resource is called and because the file isn't there, the set method is called to create it.

Running the command again:

![dscresources01](/images/2017-09/dscresres02.png){: width="600"}

As you can see, chef is handling this in a idempotent way as we know from DSC.

That's it for this post. We have seen how DSC resources can be consumed by chef and this was my main first goal. To be continued!