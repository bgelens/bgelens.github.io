---
title:  "Working with DSC in the context of 3rd party CM tooling. Part 6: Puppet intro"
date:   2018-01-21 11:30:00
categories: ["Desired State Configuration","Puppet"]
tags: ["Desired State Configuration","Puppet"]
---

In this post I start my investigation on how to get started with puppet. Again (as with the chef series), the main objective is to see how puppet can improve the experience on using DSC, so the investigation will mainly focus on the integration / interaction between the two.

To be honest with you, I knew a little bit about chef before I started working on that series. Not a lot, but enough to kick start me on that journey. Puppet however is entirely new to me. I know absolutely zero about this configuration management solution. I bet the investigation will be fun and I hope writing about my experiences will be helpful for you as well.

Index:

* [Part 1: Chef intro](http://bgelens.nl/working-with-dsc-in-the-context-of-3rd-party-cm-tooling-part-1-chef-intro)
* [Part 2: Chef notifications](http://bgelens.nl/working-with-dsc-in-the-context-of-3rd-party-cm-tooling-part-2-chef-notifications/)
* [Part 3: Chef server](http://bgelens.nl/working-with-dsc-in-the-context-of-3rd-party-cm-tooling-part-3-chef-server/)
* [Part 4: Chef bootstrapping](http://bgelens.nl/working-with-dsc-in-the-context-of-3rd-party-cm-tooling-part-4-chef-bootstrap/)
* [Part 5: Chef client settings](http://bgelens.nl/working-with-dsc-in-the-context-of-3rd-party-cm-tooling-part-5-chef-clientsettings/)
* [Part 6: Puppet intro](http://bgelens.nl/working-with-dsc-in-the-context-of-3rd-party-cm-tooling-part-6-puppet-intro/)
* [Part 7: Puppet notifications](http://bgelens.nl/working-with-dsc-in-the-context-of-3rd-party-cm-tooling-part-7-puppnotificationset/)
* [Part 8: Puppet server](http://bgelens.nl/working-with-dsc-in-the-context-of-3rd-party-cm-tooling-part-8-puppet-server/)
* [Part 9: Puppet Modules, Roles and more](http://bgelens.nl/working-with-dsc-in-the-context-of-3rd-party-cm-tooling-part-9-puppet-modules_roles_and_more/)
* [Part 10: Puppet bootstrap](http://bgelens.nl/working-with-dsc-in-the-context-of-3rd-party-cm-tooling-part-10-puppet-bootstrap/)

So let’s get started with the first post on using Puppet in combination with DSC.

# Puppet

![puppet logo](/images/2018-01/puppet-logo.png){: height="200px"}

At the heart of Puppet is the [puppet agent](https://puppet.com/docs/puppet/5.3/man/agent.html){:target="_blank"} which is the engine that does all the heavy lifting. You can compare it a bit with the Local Configuration Manager when talking about DSC. Normally it will operate in conjunction with a [puppet master](https://puppet.com/docs/puppet/5.3/man/master.html){:target="_blank"} implementation which you can view as the DSC Pull Server or Azure Automation DSC but the puppet agent can run in a standalone fashion as well. Unlike with chef, there doesn't seem to be a hosted pre-baked Puppet as a service offering so you'll have to create a puppet server yourself if you want to use puppet in a server / client model. In this post we will stick to the stand-alone model and we'll investigate the server implementation in a later post.

## Deploy a new VM in Azure

I'm going to deploy a new fresh VM for this investigation using azure cli.

```bash
az group create -l westeurope -n puppet

az vm create -n puppetdsc01 -g puppet --image MicrosoftWindowsServer:WindowsServer:2016-Datacenter-Server-Core:latest --no-wait --size Standard_DS1_V2 --admin-username ben --admin-password 'My$uperSecretPa$$w0rd'
```

Now the VM is created, let's connect to it with RDP and install the latest puppet agent.

Fetch the public IP:

```bash
az vm list-ip-addresses -n puppetdsc01 --output table
```

## Install Puppet Agent

The very first thing I'm going to do is to install the puppet agent. The Windows version of the agent can be downloaded [here](https://downloads.puppetlabs.com/windows/puppet5/).

When browsing this page, we figure out which version is the latest:

![versions](/images/2018-01/puppetagentdownload.png)

At the time of writing, version 5.3.3 is the latest. Luckily the latest version is referenced by puppet-agent-x64-latest.msi as well so for future downloads, there is no need to revisit this page as we can just target [this](https://downloads.puppetlabs.com/windows/puppet5/puppet-agent-x64-latest.msi){:target="_blank"} link.

Let's download and install the agent.

```powershell
Invoke-WebRequest -UseBasicParsing -Uri https://downloads.puppetlabs.com/windows/puppet5/puppet-agent-x64-latest.msi -OutFile c:\puppet-agent-x64-latest.msi
Unblock-File -Path C:\puppet-agent-x64-latest.msi
C:\puppet-agent-x64-latest.msi
```

![setup1](/images/2018-01/agentinstall01.png)

![setup2](/images/2018-01/agentinstall02.png)

![setup3](/images/2018-01/agentinstall03.png)

Ignore the puppet master configuration for now. I found that it is not checked if the server exists to continue the installation.

![setup4](/images/2018-01/agentinstall04.png)

![setup5](/images/2018-01/agentinstall05.png)

Now that puppet agents is installed. Let’s install vs-code so we have a nice editor available.

```powershell
Invoke-WebRequest -UseBasicParsing -Uri https://go.microsoft.com/fwlink/?Linkid=852157 -OutFile c:\vscode.exe
Unblock-File -Path c:\vscode.exe
C:\vscode.exe
```

Make sure you select the option that code would be added to the Path. Logoff and log back in again.

I’ll install the PowerShell extension and the official puppet extension (not sure yet if it will be beneficial but it will probably help!)

```powershell
code --install-extension ms-vscode.PowerShell
code --install-extension jpogran.puppet-vscode
```

# Start Small

As I did with the first post in the Chef series, I'll try to start as small as possible and work my way up from there. The smallest configuration fragment with puppet is called a [manifest](https://puppet.com/docs/puppet/5.3/lang_summary.html#files){:target="_blank"}. It is a file containing resource defintions (simular to a DSC configuration script).

Let's create a folder in the root of the c drive and call it puppet. Now open code at this folder and create a new file called ```dsc01.pp```. The pp extension stands for puppet program as far as I can tell.

Start typing a few letters of the word ```resources``` and you'll see the puppet vs code extension suggesting a code snippet to use.

![resources](/images/2018-01/resources.png)

Use tab to insert it.

![resources2](/images/2018-01/resources2.png)

Great! Now we know how a resource defintion looks like with puppet. Manifests are written in the [puppet language](https://puppet.com/docs/puppet/5.3/lang_summary.html){:target="_blank"}.

> Puppet uses its own configuration language, which was designed to be accessible to sysadmins. The Puppet language does not require much formal programming experience and its syntax was inspired by the Nagios configuration file format.

*From the puppet man pages*

```puppet
resources { 'title':
  ensure => 'present'
}
```

When investigating the snippet provided to by the vs code extension, I found the puppet man page describing [resources](https://puppet.com/docs/puppet/5.3/lang_resources.html#resource-types){:target="_blank"}.

The ```resources``` word would be replaced with a ```resource type``` (resource name). The title can be mapped to the name attribute of a resource but this isn't required (can be overwritten by specifying the attribute). The title needs to be unique within the manifest per resource type (just as with a DSC configuration).

We want to use a DSC resource so we need to find the resource type that enables this functionality. The puppet agent ships with a couple of resources of which the types can be found using ```puppet resource --types```.

![inboxtypes](/images/2018-01/inboxtypes.png)

As you can see, the inbox resource types don't show anything related to DSC. So we already know we need to expand the known resource types in some way to make use of DSC resources. So how do we extend the resource types (add more puppet resources)? You guessed it, using [modules](https://puppet.com/docs/puppet/5.3/modules_fundamentals.html){:target="_blank"}.

You can find and install modules from a place called [puppet forge](https://forge.puppet.com/){:target="_blank"}. The command line information is located [here](https://puppet.com/docs/puppet/5.3/modules_installing.html#finding-forge-modules){:target="_blank"}. Puppet forge is like the PowerShell gallery and chef supermarket. When we search for DSC, two modules by puppetlabs are what we are looking for.

![puppetforge](/images/2018-01/puppetforge.png)

* dsc
* dsc_lite

After some research I found that for me, **dsc_lite** is the way to go. Let me elaborate on why.

## dsc module

The dsc module can be found [here](https://forge.puppet.com/puppetlabs/dsc){:target="_blank"}. It is developed on [GitHub](https://github.com/puppetlabs/puppetlabs-dsc){:target="_blank"}. It is good to know that usage of this module is supported and the dsc_lite module is not.

The advantage of using this module is that the DSC resources it support (inbox and mof schema based DSC resource kit resources) are included with the module. There is no need to distribute the resources to the managed nodes via other means. Intellisense with the vs code extension is picked up as the supported DSC resources are represented as puppet resource types and therefore the required info is included. The disadvantage is that if you want to include non-included DSC resources, you need to build your own version of the module and do some work to include your resource.

## dsc_lite module

The dsc_lite module can be found [here](https://forge.puppet.com/puppetlabs/dsc_lite){:target="_blank"}. It is developed on [GitHub](https://github.com/puppetlabs/puppetlabs-dsc_lite){:target="_blank"}. This module is not supported, unlike the dsc module.

The advantage of using this module is that it works with any DSC resource (inbox, DSC resource kit, custom, class, etc.) that is installed on the managed node. You can see the dsc_lite module as the equivalent of the chef ```dsc resource resource```. Disadvantage is that you need to do more manual work to map attributes correctly.

It's always good to know a little on how things work under the covers so I looked at the module code for a bit. Both modules make use of ```Invoke-DscResource``` under the covers by mapping to the DSC ```Test``` and ```Set``` methods. The logic is defined in an ```erb``` ([Embedded Ruby Block](https://puppet.com/docs/puppet/4.9/lang_template_erb.html){:target="_blank"}) [template](https://github.com/puppetlabs/puppetlabs-dsc_lite/blob/master/lib/puppet/provider/base_dsc_lite/invoke_generic_dsc_resource.ps1.erb#L34){:target="_blank"} which is a template containing inline Ruby statements that will be executed (e.g. variable expansion or calling ruby functions) at evaluation time resulting in the desired content. The erb template is used by the puppet [PowerShell provider](https://github.com/puppetlabs/puppetlabs-dsc_lite/blob/master/lib/puppet/provider/base_dsc_lite/powershell.rb){:target="_blank"} contained in the module.

Let's install the dsc_lite module and see how we make use of it.

First, let's see what modules are installed by default:

```bash
puppet module list
```

![installedbydefault](/images/2018-01/modulesinstalledbydefault.png)

Now let's find the module we want via the command line.

```bash
puppet module search dsc
```

![searchmodule](/images/2018-01/cmdlinesearch.png)

Now we know what name to use, let's install the lite module.

```bash
puppet module install puppetlabs-dsc_lite
```

![installmodule](/images/2018-01/moduleinstall.png)

From the looks of it, the module install command also installs dependent modules. When checking the module [metadata.json](https://github.com/puppetlabs/puppetlabs-dsc_lite/blob/master/metadata.json){:target="_blank"}, you can see that dependencies are governed there.

![metadata](/images/2018-01/metadata.png)

Now we need to tell the puppet vs code extension to restart (I noticed that it uses a cache so you won't see it providing anything you just installed before you reload it).

![restartsession](/images/2018-01/restartpuppetext.png)

And now we are finally ready to create that first puppet manifest to consume a dsc resource. I'll go with the same example I used for Chef and make use of the inbox dsc file resource.

First let's type the word ```dsc``` to see if the extension gives us something.

![dscsnippet](/images/2018-01/dscsnippet.png)

Cool! Now let's see if is able to give us some insight on the attributes to use (you could of course just read the [documentation](https://forge.puppet.com/puppetlabs/dsc_lite#usage){:target="_blank"} but I think helpful tooling really speeds up the learning).

![attributecompletion](/images/2018-01/attributecompletion.png)

This extension is proving it's worth right from the start! Let's look at a completed resource definition.

```puppet
dsc { 'some file':
  dsc_resource_module_name => 'PSDesiredStateConfiguration',
  dsc_resource_name => 'File',
  dsc_resource_properties => {
    ensure => 'present',
    destinationpath => 'c:/testfile.txt',
    contents => 'some text',
  }
}
```

**NOTE: Be sure to specify all DSC resource property names in lower casing or else puppet won't find the types**

Let's apply this manifest. In a standalone implementation a manifest can be applied using the puppet [apply](https://puppet.com/docs/puppet/5.3/man/apply.html){:target="_blank"} sub command.

First let's try ```--noop``` mode which basically is a ```WhatIf```.

```bash
puppet apply .\dsc01.pp --noop
```

![noop](/images/2018-01/noop.png)

As you can see, puppet is telling you what would be done when run without the ```noop``` parameter specified.

Now let's run it without the ```noop``` parameter.

```bash
puppet apply .\dsc01.pp
```

![created](/images/2018-01/created.png)

If you want more verbose output you can add the ```--debug``` parameter.

That’s it for this post. We have seen how DSC resources can be consumed by puppet and this was my main first goal. To be continued!
