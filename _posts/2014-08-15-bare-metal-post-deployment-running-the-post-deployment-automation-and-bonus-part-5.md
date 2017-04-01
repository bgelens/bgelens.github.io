---
title:  "Bare Metal Post-Deployment – Running the Post-Deployment Automation and Bonus – Part 5"
date:   2014-08-15 12:00:00
categories: ["Hyper-V","SCVMM","Bare Metal Deployment","SMA","Windows Server","Virtual Fibre Channel"]
tags: ["Hyper-V","SCVMM","Bare Metal Deployment","SMA","Windows Server","Virtual Fibre Channel"]
---
This blog post is written and published on Hyper-V.nu: [https://hyper-v.nu/archives/bgelens/2014/08/bare-metal-post-deployment-running-the-post-deployment-and-bonus-part-5/](https://hyper-v.nu/archives/bgelens/2014/08/bare-metal-post-deployment-running-the-post-deployment-and-bonus-part-5/){:target="_blank"}

This blog post series consists of 5 parts:

1. [Part 1: Introduction and Scenario](http://bgelens.nl/bare-metal-post-deployment-introduction-and-scenario-part-1/)
2. [Part 2: Pre-Conditions  (this blog post)](http://bgelens.nl/bare-metal-post-deployment-pre-conditions-part-2/)
3. [Part 3: Constructing the Post-Deployment automation](http://bgelens.nl/bare-metal-post-deployment-constructing-the-post-deployment-automation-part-3/)
4. [Part 4: Constructing the Post-Deployment automation continued](http://bgelens.nl/bare-metal-post-deployment-constructing-the-post-deployment-automation-continued-part-4/)
5. [Part 5: Running the Post-Deployment automation and Bonus](http://bgelens.nl/bare-metal-post-deployment-running-the-post-deployment-automation-and-bonus-part-5/)

# Running the Post-Deployment automation

So you have arrived at the last blog post in this series. Hope you have enjoyed everything you have read and must importantly have learned what you were seeking to learn. As a little extra, I’ve put in a little bonus script to finalize a Hyper-V cluster configuration.
When everything is in place we simply run the “Run-HyperVPostdeployment” runbook.

![](/images/2014-08/081214_1007_BareMetalPo1.png)

Output of child runbooks is returned to the master runbook by using return statements and sending them back as strings. You only have to check the job summery of the master runbook to get a view of how things went.

![](/images/2014-08/081214_1007_BareMetalPo2.png)

When things didn’t go well, you can see at which stage the failure occurred and start troubleshooting from there. Check the VMM job log to get more detailed information.

![](/images/2014-08/081214_1007_BareMetalPo3.png)

I didn’t include cleanup steps but it’s really easy to restart the process by removing all logical switches from the host and just restart the master Runbook.

# Bonus Post Cluster Deployment Configuration script

Run the following script on one of the Cluster nodes (this has to run only once per cluster). The script will:

* Configure the cluster to register its PTR records into the reverse lookup zone.
* Rename the cluster networks
* Configure the correct cluster network metrics
* Configure the networks allowed for live migration
* Configure 512MB RAM as CSV block cache
* Configure the SameSubnetThreshold for 20 seconds
* Configure the cluster service shutdown timeout to 30 minutes
* Renames the CSV mount point to reflect the volume name
* Remove the quorum drive letter

```powershell
$ManagementNetwork = "10.10.9.*"
$LivemigrationNetwork = "10.10.10.*"
$CSVNetwork = "10.10.11.*"
$cluster = (get-cluster).name
Get-ClusterResource -Cluster $cluster -Name "Cluster Name" | set-ClusterParameter -Name PublishPTRRecords -Value 1
Get-ClusterResource -Cluster $cluster -Name "Cluster Name" | Stop-ClusterResource
Get-ClusterResource -Cluster $cluster -Name "Cluster Name" | Start-ClusterResource
$clusterresource = Get-ClusterResourceType -Name "Virtual Machine" -Cluster $cluster
Get-ClusterNetwork -Cluster $cluster | %{
    if ($_.address -like $ManagementNetwork)
        {
        write-host $_ is MGMT
        $_.role = 3
        $exclude1 = $_.id
            if ($_.name -ne "MGMT")
                {
                $_.name = "MGMT"
                }

        }

    elseif ($_.address -like $LivemigrationNetwork)
        {
        write-host $_ is LM
        $_.Metric = 200
        $_.role = 0
            if ($_.name -ne "LM")
                {
                $_.name = "LM"
                }
        }

    elseif ($_.address -like $CSVNetwork)
        {
        write-host $_ is CSV
        $_.role = 1
        $_.Metric = 100
        $exclude2 = $_.id
            if ($_.name -ne "CSV")
                {
                $_.name = "CSV"
                }
        }
    else
        {
        # do nothing
        }
    $clusterresource | Set-ClusterParameter -Name MigrationExcludeNetworks -Value ($exclude1 + ";" + $exclude2)
}
(get-cluster $cluster).blockcachesize = 512
(Get-Cluster).SameSubnetThreshold = 20
(get-cluster).ShutdownTimeoutInMinutes = 30

Get-CimInstance win32_volume -Filter 'filesystem="CSVFS"' | %{
    Rename-Item -Path $_.name -NewName $_.label
}
$q = (Get-ClusterQuorum).quorumresource
$session = New-CimSession -ComputerName $q.ownergroup.ownernode.name
$volumeguid = (Get-CimInstance -Namespace "root\MSCluster" -class "MSCluster_DiskPartition" -CimSession $session | ?{$_.path -notlike "\\*"}).volumeguid
Get-CimInstance -ClassName win32_volume -CimSession $session | ?{$_.DeviceID -eq "\\?\Volume{$volumeguid}\"} | Set-CimInstance -Arguments @{driveletter=$null}
$session | Remove-CimSession
```

Please find a link to all script files [here](https://onedrive.live.com/redir?resid=7F054A45CE77AA30!18768&authkey=!AFTXnyN9HcygdC4&ithint=folder%2c%20){:target="_blank"}.