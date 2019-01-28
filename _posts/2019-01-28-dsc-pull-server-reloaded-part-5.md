---
title:  "DSC Pull Server reloaded. Part 5: Containers!"
date:   2019-01-28 12:00:00
categories: ["Desired State Configuration","Pull Server","Container"]
tags: ["Desired State Configuration","Pull Server","Container"]
excerpt_separator: <!--more-->
---

With Windows Server 2016, container support was introduced. Now that the Pull Server supports SQL, it felt naturally to me to try and see if the Pull Server could be containerized.
A containerized Pull Server allows us to run a Pull Server anywhere where Windows Server Containers are supported (also on Windows 10!). Let's investigate a prototype!

> On PSConfAsia and PSConfEU I demoed the Pull Server running in containers. Please see the video section for the session recordings.

<!--more-->

Index:

* [Part 1: SQL Support](https://bgelens.nl/dsc-pull-server-reloaded-part-1-sql-support)
* [Part 2: Managing the Pull Server with DSCPullServerAdmin](https://bgelens.nl/dsc-pull-server-reloaded-part-2-managing-the-pull-server-with-dscpullserveradmin)
* [Part 3: Pre-create the Pull Server database](https://bgelens.nl/dsc-pull-server-reloaded-part-3-precreate-pull-server-database)
* [Part 4: Migrate EDB or MDB to SQL](https://bgelens.nl/dsc-pull-server-reloaded-part-4-migrate-edb-mdb-to-sql)
* Part 5: Containers! (this post)

# Environment

For this blog post I'm running everything on my laptop which is running Windows 10. I installed Hyper-V and the Containers feature as well as Docker for Desktop. Docker for Desktop is running in Windows Containers mode. To get started please follow the [Microsoft documentation](https://docs.microsoft.com/en-us/virtualization/windowscontainers/quick-start/quick-start-windows-10){:target="_blank"}.

![dockerversion](/images/2019-01/dockerversion.png)

I've pre-pulled the Windows Server Core 2019 Windows Container image that I'll use for building my Pull Server image.

```powershell
docker pull mcr.microsoft.com/windows/servercore:ltsc2019
```

![dockerimage](/images/2019-01/dockerimage.png)

# Docker project

I've created a folder containing a couple of files to setup my container image.

![dockerfiles](/images/2019-01/dockerfiles.png)

## DockerFile

The DockerFile contains the build instructions for the Docker Image.

```docker
FROM mcr.microsoft.com/windows/servercore:ltsc2019
LABEL maintainer="ben@bgelens.nl"
COPY [ "Docker.ps1", "./Docker.ps1" ]
COPY [ "DockerMon.ps1", "./DockerMon.ps1" ]
RUN powershell.exe -Command .\Docker.ps1
EXPOSE 8080
CMD ["powershell.exe", "c:\\DockerMon.ps1"]
```

As you can see, we are basing on the `ltsc2019` version of the `servercore` image. We are copying the other files from the directory. The `Docker.ps1` script is executed and finally some metadata is added to the resulting image, telling you that it is expected to bind to the container local port `8080` and when running a container based on this image, the `DockerMon.ps1` script is executed.

## Docker.ps1

The Docker.ps1 script contains all the setup logic for the container image.

```powershell
$null = Install-WindowsFeature DSC-Service
$null = Install-PackageProvider -Name PowerShellGet -ForceBootstrap -Force
$null = Install-Module -Name xPSDesiredStateConfiguration -RequiredVersion 8.4.0.0 -Force
$null = Invoke-DscResource -ModuleName xPSDesiredStateConfiguration -Name xDscWebService -Method Set -Property @{
    Ensure = 'Present'
    EndpointName = 'PSDSCPullServer'
    Port = 8080
    PhysicalPath = "$env:SystemDrive\inetpub\PSDSCPullServer"
    CertificateThumbPrint='AllowUnencryptedTraffic'
    ModulePath="$env:PROGRAMFILES\WindowsPowerShell\DscService\Modules"
    ConfigurationPath="$env:PROGRAMFILES\WindowsPowerShell\DscService\Configuration"
    State='Started'
    RegistrationKeyPath="$env:PROGRAMFILES\WindowsPowerShell\DscService"
    AcceptSelfSignedCertificates=$true
    UseSecurityBestPractices=$false
    SqlProvider=$true
    SqlConnectionString='#CONNECTIONSTRING#'
}
$null = Remove-Website -Name 'Default Web Site'
$null = Stop-Website -Name PSDSCPullServer
```

As you can see, the DSC-Service feature is installed, the `xPSDesiredStateConfiguration` module is installed and the `xDscWebService` DSC resource is executed to setup the Pull Server. A placeholder is used for the Sql Connection String. Finally the Pull Server is stopped as when we start a new container based on the image we are producing, we want the web.config changed to be applied from the start.

## DockerMon.ps1

The DockerMon.ps1 script contains the logic to start the container using the correct specified settings (passed in via Environmental variables) and verify that the container is operating properly (or else terminating it).

```powershell
$configFile = 'C:\inetpub\PSDSCPullServer\web.config'
$connString = $env:ConnectionString
if ($null -eq $connString) {
    # could also result in edb usage instead of sql so container can run in solo mode
    Write-Error -Message 'No ConnectionString found in Environment variables' -ErrorAction Stop
}
$webConfig = Get-Content -Path $configFile
$webConfig = $webConfig.replace('#CONNECTIONSTRING#', $connString)
if (Test-Path -Path c:\pullserver) {
    # if it doesn't keep defaults to at least be able to run
    $webConfig = $webConfig.replace('C:\Program Files\WindowsPowerShell\DscService', 'c:\pullserver')
}
$webConfig | Set-Content $configFile

Start-Website -Name PSDSCPullServer

function Monitor {
    $irmArgs = @{
        Headers = @{
            Accept = 'application/json'
            ProtocolVersion = '2.0'
        }
        UseBasicParsing = $true
        Uri = 'http://localhost:8080/PSDSCPullServer.svc'
        ErrorAction = 'Stop'
    }
    try {
        $null = Invoke-RestMethod @irmArgs
        "$([datetime]::UtcNow.ToString()) - Pull Server is responding"
        Start-Sleep -Seconds 10
        Monitor
    } catch {
        throw $_
    }
}

Monitor
```

As you can see, when no `ConnectionString` is found in the Environment variables, the container is terminated immediately. When it is found, the ConnectionString placeholder in the web.config file is replaced with the desired value.

When the folder `c:\pullserver` is found, all references in the web.config file that refer to `C:\Program Files\WindowsPowerShell\DscService` are replaced with `c:\pullserver`. For this prototype it is expected that a volume is mounted to the container (for persistent storage) containing the configurations, modules and registrationkeys.

The Pull Server website is started.

A self invoking monitor function is started that checks every 10 seconds if the internal Pull Server endpoint is responding (the Pull Server will throw 500 errors when the Database is not reachable for whatever reason). When something is wrong, the process is terminated resulting in a stopped container. In a more advanced usage scenario (e.g. Swarm or Kubernetes) this will result in the scheduler to start a new instance of the container knowing something has gone wrong.

# Building the image

Now we have seen the files involved, we can start building the container image.

```docker
docker build . -t pullserver:latest
```

![dockerbuild](/images/2019-01/dockerbuild.png)

Build was successful. Let's see if it's available:

![imagecreated](/images/2019-01/imagecreated.png)

# Running a Pull Server Container

To run the container we need a database pre created (or an existing one), and a directory structure containing configuration files, module archives, checksums and registrationkey files.

>When setup correctly you can see we could scale out to multiple container relatively easily. The database already is easily accessible and sharable by multiple container instances and the directory structure could be made highly available as well which allows you to map it into multiple containers.

## Prerequisites

I have created a new database as Azure SQL database already by using `DSCPullServerAdmin` Module. See [part 3](https://bgelens.nl/dsc-pull-server-reloaded-part-3-precreate-pull-server-database) of this series for the instructions on how to do this.

The directory which I will map into the container looks like this:

![pullserverfiles](/images/2019-01/pullserverfiles.png)

I just used a simple configuration to generate a mof. The configuration relies on a DSC Resource Module, so I included that one as well. I won't go into detail for the configuration as it's not really important for this blog post.

## Starting and checking the container

We now start a container based from the image passing in what is needed.

```docker
docker run `
    -p 8080:8080 `
    -v C:\pullserver:C:\pullserver `
    -e ConnectionString="Provider=SQLOLEDB.1;Server=dscpullsrv.database.windows.net;Database=pullsrvdb;User ID=myUserName;Password=myPassword;" `
    -d pullserver
```

A couple of things to note here:

* we bound port 8080 of the local machine to port 8080 of the container by specifying `-p 8080:8080`
* we mapped the directory c:\pullserver to the container path c:\pullserver by specifying `-v C:\pullserver:C:\pullserver`
* we passed in the environmental variable ConnectionString by specifying `-e ConnectionString="Provider=SQLOLEDB.1;Server=dscpullsrv.database.windows.net;Database=pullsrvdb;User ID=myUserName;Password=myPassword;"`
* we specified that the container should be running detached from the terminal by specifying `-d`

Let's see if the container is still running.

```docker
docker ps
```

![containerrunning](/images/2019-01/containerrunning.png)

So it is running, we could inspect the logs to see if we get some expected output there.

```docker
docker logs 96
```

![dockerlogs](/images/2019-01/dockerlogs.png)

So it seems the Pull Server is operating in the container! Let's onboard a machine to it to see if registration will work out as well.

>For me, the localhost mapping did not result in connectivity to the container. As I'm running an edge build of Docker for Desktop and a Windows Insider build it's probably something specific to my setup. To work around the issue, I'll connect the LCM using the container IP instead of localhost. You can find the IP address by using `docker inspect` or `docker exec`.

```powershell
[dsclocalconfigurationmanager()]
configuration lcm {
    Settings {
        RefreshMode = 'Pull'
    }

    ConfigurationRepositoryWeb SQLPullWeb {
        ServerURL = 'http://172.17.18.205:8080/PSDSCPullServer.svc'
        RegistrationKey = 'cb30127b-4b66-4f83-b207-c4801fb05087'
        AllowUnsecureConnection = $true
    }
}

lcm
Set-DscLocalConfigurationManager .\lcm -Verbose
```

![onboard](/images/2019-01/onboard.png)

Onboarding worked as expected! Now let's see if we can pull the configuration.

![update](/images/2019-01/update.png)

Awesome! Prototype successful!

>If you want to cleanup your running Pull Server container run: `docker rm -f (docker ps -aq)`
