---
title:  "Working with DSC in the context of 3rd party CM tooling. Part 4: Chef bootstrapping"
date:   2017-11-28 16:30:00
categories: ["Desired State Configuration","Chef"]
tags: ["Desired State Configuration","Chef"]
---

In this post I'll continue my investigation on using DSC with Chef. In this post I'll look at how we can onboard new nodes more easily (without the need to log into them interactively) and have them apply a run-list immediately (have the initial convergence to be part of the Chef agent install or VM deployment process). There won't be much DSC content in this post but it is good to know how to get your nodes into management and have a configuration bootstrapped!

Index:

* [Part 1: Chef intro](http://bgelens.nl/working-with-dsc-in-the-context-of-3rd-party-cm-tooling-part-1-chef-intro)
* [Part 2: Chef notifications](http://bgelens.nl/working-with-dsc-in-the-context-of-3rd-party-cm-tooling-part-2-chef-notifications/)
* [Part 3: Chef server](http://bgelens.nl/working-with-dsc-in-the-context-of-3rd-party-cm-tooling-part-3-chef-server/)
* [Part 4: Chef bootstrapping](http://bgelens.nl/working-with-dsc-in-the-context-of-3rd-party-cm-tooling-part-4-chef-bootstrap/)
* [Part 5: Chef client settings](http://bgelens.nl/working-with-dsc-in-the-context-of-3rd-party-cm-tooling-part-5-chef-clientsettings/)
* [Part 6: Puppet intro](http://bgelens.nl/working-with-dsc-in-the-context-of-3rd-party-cm-tooling-part-6-puppet-intro/)
* [Part 7: Puppet notifications](http://bgelens.nl/working-with-dsc-in-the-context-of-3rd-party-cm-tooling-part-7-puppnotificationset/)
* [Part 8: Puppet server](http://bgelens.nl/working-with-dsc-in-the-context-of-3rd-party-cm-tooling-part-8-puppet-server/)

We'll look at two scenario's:

1. Existing node which is not under Chef management yet
2. Brand new node which onboards during deployment

# Deploy a new VM

For the first scenario, let's deploy a new VM using Azure CLI. This time I'm deploying the 1709 SA build of Windows Server.

```sh
az vm create -n chefdsc02 -g chefdsc --image MicrosoftWindowsServer:WindowsServerSemiAnnual:Datacenter-Core-1709-smalldisk:1709.0.20171012 --no-wait --size Standard_DS1_V2 --admin-username ben --admin-password 'My$uperSecretPa$$w0rd'
```

I already had the *chefdsc* resource group present, if you don't have that anymore, please look at the first post in this series to see how it was created.

## Bootstrap the node using knife

Make sure you navigate to your chef directory where the .chef subdirectory is located containing the knife.rb file.

Once the VM is created we are going to remotely "bootstrap" it with the Chef client. Meaning, the client will be installed and the client will be onboarded into the Chef server during the same process. Once onboarded the client will execute the assigned run-list as well. To do this we'll make use of the *knife* tool from the ChefDK once again. This time we'll make use of the [knife windows plugin](https://docs.chef.io/plugin_knife_windows.html){:target="_blank"} which is needed to handle Windows oriented operations. Luckily for us, the knife windows plugin is shipped by default as part of the ChefDK nowadays, so no need to install it manually.

If you want to see you have it installed, run the following:

```bash
chef gem list win
```

![gems](/images/2017-11/gems.png){: width="600px"}

We are going to connect to the VM over the public ip address which was deployed as part of the ```az vm create``` command.
To figure out which public ip address was created run the following:

```bash
az vm list-ip-addresses -g chefdsc -n chefdsc02 -o table
```

![publicip](/images/2017-11/publicip.png){: width="600px"}

Now we know the public ip of the VM, let's see if knife is able to talk to the vm using it. Note that I'm specifying the ```--manual-list``` argument. This will tell knife that the node shouldn't be looked up on the chef server (a lookup is the default behavior, this won't work now as the node is not under management yet).

```bash
knife wsman test 52.178.77.145 --manual-list
```

![knifefailure](/images/2017-11/knifefailure.png){: width="600px"}

Doesn't look good. We need to open the winrm tcp port 5985 on the NSG. By default ```az vm create``` only opens rdp for Windows and ssh for Linux VMs.

```bash
az vm open-port --port 5985 --resource-group chefdsc --name chefdsc02
```

Let's try again.

```bash
knife wsman test 52.178.77.145 --manual-list
```

![knifefailure](/images/2017-11/knifefailure.png){: width="600px"}

Still not good. Since I know my PowerShell remoting I already know that the VM will have the Windows Firewall Public profile active and inbound winrm is not allowed in that case.

I'm trying to illustrate it's not that straight forward to have a "zero-touch" bootstrap happening like this. You need to consider a few things to allow for this to happen more smoothly.
Luckily we can fix this nowadays using a new azure functionality *run-command* which allow you to run commands in the VM without the need to fully define a script extension resource.

To see which run-command types are available:

```bash
az vm run-command list --location westeurope --query "[*].{commandid:id,label:label}"
```

![runcommandshow](/images/2017-11/runcommandshow.png){: width="600px"}

We are only targetting to open the winrm port on the public profile so we won't choose the *EnableRemotePS* command type. Instead we will use the *RunPowerShellScript* type.

```bash
az vm run-command invoke --resource-group chefdsc --name chefdsc02 --command-id RunPowerShellScript --scripts "Set-NetFirewallRule -Name WINRM-HTTP-In-TCP-PUBLIC -RemoteAddress any"
```

![runcommandinvoke](/images/2017-11/runcommandinvoke.png){: width="600px"}

Let's verify if knife is able to communicate with the VM now.

![knifesuccess](/images/2017-11/knifesuccess.png){: width="600px"}

Victory! Finally everything is in place to make this "bootstrap" happening. It is probably easier to define everything as an ARM template and have the prerequisites configured as part of its deployment making use of the script extension or even better, the Chef extension! In the next paragraph the Chef extension will be used to show how much easier it all is. For now, let's continue the bootstrap using knife so you'll appreciate the Chef extension more.

Knife needs to be able to talk to the node directly to bootstrap it. This works fine when you run knife inside of the boundaries of a fully routed network but doesn't work well with segregated / isolated networks. The reason I point this out is, it's just one of the ways to handle this and not the only one.

Let's start the bootstrap procedure by telling the bootstrap sub-command to make use of winrm instead of ssh and provide the necessary authentication data as well as an initial run list.

```bash
knife bootstrap windows winrm 52.178.77.145 --winrm-user ben --winrm-password 'My$uperSecretPa$$w0rd' --node-name chefdsc02 --run-list 'recipe[dsctest01]'
```

![knifebootstrapresult](/images/2017-11/knifebootstrapresult.png){: width="600px"}

Above is a cutout from the entire output stream. Basically what is happening is that a node object is created on the chef server, the chef client is downloaded and installed and a chef run is started with the run list specified. Nice!

This chef client is however not configured to run periodically. It is not installed as a service and no scheduled task has been registered to trigger it after a certain amount of time. We'll look into how to configure this a little bit later (first we'll look at using the Chef VM extension).

# Deploy a new VM with Chef extension

This time I'll deploy a VM without a public ip and add the Chef VM extension to have it onboard and converge.

First, let's create the VM (note I'm using Server 2016 instead of 1709 due to some issues I had with the latter):

```bash
az vm create -n chefdsc03 -g chefdsc --image MicrosoftWindowsServer:WindowsServer:2016-Datacenter:latest --no-wait --size Standard_DS1_V2 --admin-username ben --admin-password 'My$uperSecretPa$$w0rd' --public-ip-address ''
```

Note the ```--public-ip-address ''``` argument. This will *prevent* a new public ip resource to be created and associated.

Now let's add the [Chef VM extension](https://github.com/chef-partners/azure-chef-extension/blob/master/README.md){:target="_blank"}.

We need to find the correct one first.

```bash
az vm extension image list --latest --location westeurope --name chefclient
```

![listextension](/images/2017-11/chefvmextensionlist.png){: width="600px"}

As you can see, there is a linux and a windows version of the extension. Note the ```--latest``` argument which will only list the latest version.

Now we will add the VM extension to the VM. Special thanks to [Stuart Preston](https://twitter.com/stuartpreston){:target="_blank"} for giving me the hint on using eval to get this to work!

```bash
#base64 encode user.pem (replace with your own pem, this could also be the orgvalidator.pem)
validationkeyb64=$(cat .chef/bgelens.pem | base64 -w 0)

#chef url
chefurl=$(cat .chef/knife.rb  | grep chef_server_url | tr -s ' ' | cut -d ' ' -f 2 | tr -d '"')

chefsettings=$(printf '{"bootstrap_options": { "chef_node_name":"chefdsc03","chef_server_url":"%s", "validation_client_name":"bgelens"},"daemon": "none","validation_key_format": "base64encoded","runlist":"recipe[dsctest01]"}' $chefurl)

protectedsettings=$(printf '{ "validation_key": "%s"}' $validationkeyb64)

eval "az vm extension set --name ChefClient --publisher Chef.Bootstrap.WindowsAzure --resource-group chefdsc --vm-name chefdsc03 \
    --settings '$chefsettings' \
    --protected-settings '$protectedsettings'"
```

![knifebootstrapresult](/images/2017-11/chefvmextensionprov.png){: width="600px"}

Please note that the success state of provisioning the Chef VM extension isn't linked to a success state of a Chef run. So success here, doesn't mean the node onboarded or ran any recipe successfully. We need to check the Chef server if the node is onboarded.

```bash
knife node list
```

![chefdsc03](/images/2017-11/chefdsc03.png){: width="600px"}

```bash
![chefdsc03-2](/images/2017-11/chefdsc03-02.png){: width="600px"}
```

Now how do we know if it ran successfully? We could go to the Chef portal but we could of course use knife for this as well.

```bash
knife node show chefdsc03
```

![chefdsc03status](/images/2017-11/chefdsc03status.png){: width="600px"}

Now if we want to see the report of the last run, we need to install a knife plugin which is not shipped with the ChefDK by default: [knife reporting](https://docs.chef.io/plugin_knife_reporting.html){:target="_blank"}.

```bash
chef gem install knife-reporting
```

![reportgeminstall](/images/2017-11/reportgeminstall.png){: width="600px"}

Once installed we need to figure out the run id of the last run (note that without installing the plugin, the runs sub command is not available).

```bash
knife runs list chefdsc03
```

![reportrunslist](/images/2017-11/reportrunslist.png){: width="600px"}

Finally we can see the entire report.

```bash
knife runs show 11e5f372-0417-442d-acd5-018befb9e39e
```

![runreport](/images/2017-11/runreport.png){: width="600px"}

Nice! That is it for this post. To be continued!
