---
title:  "Working with DSC in the context of 3rd party CM tooling. Part 2: Chef Notifications"
date:   2017-10-24 16:30:00
categories: ["Desired State Configuration","Chef"]
tags: ["Desired State Configuration","Chef"]
---

Now we have seen how to make use of a DSC resource inside of a Chef recipe, I thought of investigating a feature of Chef I heard about which is currently impossible to accomplish with native DSC without providing an intermediate state to the LCM.

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

The scenario is, something changes in one resource and to effectuate the change, a service or process needs to cycle / reload (e.g. the configuration file of the docker daemon is changed and now the docker daemon needs to restart).

When delivering this change with DSC, you would have 3 / configuration documents:

* the original state with the old desired configuration
* the new intermediate state with the change to the configuration file and the docker daemon stopped
* the final desired state with the change to the configuration file and the docker daemon started

To circumvent the need for an intermediate configuration state, I see people creating their own version of DSC resources, like the file resource, which flag the LCM that a reboot is required whenever the set method is called. This is of course not a good practice although understandable. It's an immaturaty issue that should be solved so please vote [here](https://windowsserver.uservoice.com/forums/301869-powershell/suggestions/11088639-enable-service-restart-and-similar-scenarios-in-ds){:target="_blank"}!

With Chef there is no need to define an intermediate state or ask for a reboot to effectuate. Instead we can make use of the [subscribe](https://docs.chef.io/resource_common.html#subscribes){:target="_blank"} and [notify](https://docs.chef.io/resource_common.html#notifies){:target="_blank"} resource properties which are [common functions](https://docs.chef.io/resource_common.html){:target="_blank"} for all Chef resources (like PSDscRunAsCredential is a common property for a DSC resource).

As an example, let's look at the previous recipe.

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

Let's pretent that *c:\testfile.txt* is such a configuration file which determines a daemons' behavior. First let's define the service in our recipe making use of [windows_service](https://docs.chef.io/resource_windows_service.html){:target="_blank"} Chef resource. We'll pretent that the **bits** service is impacted on the file change.

```ruby
windows_service 'bits' do
    action :start
    startup_type :automatic
end

dsc_resource 'some file' do
    resource :File
    module_name 'PSDesiredStateConfiguration'
    module_version '1.1'
    property :DestinationPath, 'c:\testfile.txt'
    property :Contents, 'some text'
    property :Ensure, 'Present'
end
```

The configuration of the ```windows_service``` resource, the **bits** service should be **started** when on evaluation it is in the stopped state for some reason. And the service should be **started** when the system is booted by setting the startup type to **Automatic**.

Now we introduce the scenario where the **bits** service should be **restarted** when the ```dsc_resource``` overwrites the *c:\testfile.txt* file induced by drift or change. We can make the ```windows_service``` resource listen for such an occurance and have it subscribe to the ```dsc_resource``` change event.

```ruby
windows_service 'bits' do
    action :start
    startup_type :automatic
    subscribes :restart, 'dsc_resource[some file]', :immediately
end

dsc_resource 'some file' do
    resource :File
    module_name 'PSDesiredStateConfiguration'
    module_version '1.1'
    property :DestinationPath, 'c:\testfile.txt'
    property :Contents, 'some text'
    property :Ensure, 'Present'
end
```

Let's remove the *c:\testfile.txt* to create configuration drift and run this recipe.

```powershell
rm c:\testfile.txt
chef-client --local-mode .\dsc01.rb
```

![subscribe](/images/2017-10/dscrecipe01.png){: width="700"}

As you can see in the screenshot, the bits service was in the desired state at the start of the recipe execution. Then the ```dsc_resource``` resource had a change event as the file was missing. The ```windows_service``` resource picked up this event and restarted the service. Very cool!

Now let's change the scenario a tiny bit and solve it using the **notifies** resource property instead. The scenario will be that the service needs to be stopped before the change to the text file can be made (e.g. due to a file lock).

```ruby
windows_service 'bits' do
    action :start
    startup_type :automatic
end

dsc_resource 'some file' do
    resource :File
    module_name 'PSDesiredStateConfiguration'
    module_version '1.1'
    property :DestinationPath, 'c:\testfile.txt'
    property :Contents, 'some text2'
    property :Ensure, 'Present'
    notifies :stop, 'windows_service[bits]', :before
    notifies :start, 'windows_service[bits]', :immediately
end
```

Instead of the ```windows_service``` resource subscribing to a change event, we now notify it of a change pending, on which it needs to stop the service. Once the change event of the ```dsc_resource``` resource is done, we will notify the ```windows_service``` resource again to start the service.

The change event is happening because I changed the file content a bit. Let's have a look at what will happen.

![notify](/images/2017-10/dscrecipe02.png){: width="700"}

As you can see, the bits service was already in the desired state. Then, the ```dsc_resource``` found out that it needed to call into the set method but first notified the ```windows_service``` resource to stop the service. It than ran the set method and after that was done, the ```windows_service``` resource was notified again that it could start the service again. Nice!

I'll stick with Chef for the next couple of blogs. To be continued!
