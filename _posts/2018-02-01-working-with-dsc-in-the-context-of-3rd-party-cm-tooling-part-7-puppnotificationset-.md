---
title:  "Working with DSC in the context of 3rd party CM tooling. Part 7: Puppet notifications"
date:   2018-02-01 21:00:00
categories: ["Desired State Configuration","Puppet"]
tags: ["Desired State Configuration","Puppet"]
---

Like Chef, Puppet also support the concept of [notifications](https://puppet.com/docs/puppet/5.3/lang_relationships.html#refreshing-and-notification){:target="_blank"}. A change  event on one resource triggers an action on another resource. Support for this kind of functionality helps with keeping resources monotonic and within the boundaries of the technology domain that is being governed by the resource. For example: a **Website** resource doesn't have to restart a *service*, as the **service** resource can be notified of the change event that requires the restart and handles that job that obviously belongs to the *service* resource domain. It also keeps you from writing custom resources when an existing resource doesn't do that one thing you need but could be handled using a notification to another resource.

I detailed some of my scenario's in the [Chef part 2](http://bgelens.nl/working-with-dsc-in-the-context-of-3rd-party-cm-tooling-part-2-chef-notifications/) post. For this post I'll investigate how we to achieve a similar implementation with Puppet.

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

Let's continue with the same manifest where we left off in the previous post.

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

The scenario again is that when the file content is changed by the resource by an update or corrective action, that the bits service needs to restart. The scenario isn't exactly useful but it still proves the point without the system needing to reboot or crash or anything.

With Puppet, the refresh action is available to a only [a couple](https://puppet.com/docs/puppet/5.1/lang_relationships.html#refreshing-and-notification){:target="_blank"} of inbox resources like the one we want to target [**Service**](https://puppet.com/docs/puppet/5.1/type.html#service){:target="_blank"}.

First let's update the manifest so it is ensured that the bits service is started and will start automatically (**enable**) when Windows is started.

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

Service { 'bits':
    ensure => 'running',
    enable => true,
}
```

Lets stop the bits service and set the startup type to manual so we can actually see the resource doing something.

```powershell
Get-Service -Name bits | Stop-Service -PassThru | set-service -StartupType Manual
```

![bitsstatus](/images/2018-02/bitsstatus.png)

Now let's apply the manifest.

```bash
puppet apply .\dsc01.pp
```

![puppetapply01](/images/2018-02/puppetapply01.png)

*Note that you can always add ```--debug``` so you can see in detail what is going on under the covers*

As the result we can inspect the service again and indeed, it now has the state of started and the startup type of automatic.

![bitsstatus02](/images/2018-02/bitsstatus02.png)

Now let's see how we implement this notification. This can be done via multiple ways, in this case it makes sense to notify the Service resource when the file changes (you could also subscribe the Service resource on file changes).

```puppet
dsc { 'some file':
  dsc_resource_module_name => 'PSDesiredStateConfiguration',
  dsc_resource_name => 'File',
  dsc_resource_properties => {
    ensure => 'present',
    destinationpath => 'c:/testfile.txt',
    contents => 'some text',
  },
  notify => Service['bits'],
}

Service { 'bits':
    ensure => 'running',
    enable => true,
}
```

As you can see, the notify takes a string in the form: ```ResourceType['name']``` similar as how DSC DependsOn is specified.

Time to introduce a change to the file.

```powershell
'File Changed!' > c:\testfile.txt
```

Now let's see if we can run the manifest in **WhatIf** (```-noop```) mode to see what would happen.

![whatif](/images/2018-02/whatif.png)

Cool! As you can see, applying this in ```noop``` mode will (as expected) give you what would happen without it being effected. Now let's apply this.

![notify](/images/2018-02/restart.png)

Voila, that was easy. I'm so hoping this functionality will show up natively in DSC one day.

To be continued!
