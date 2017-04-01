---
title:  "Testing ODX and need to switch it on and off all the time?"
date:   2013-06-12 12:00:00
categories: ["Hyper-V","Windows Server"]
tags: ["Hyper-V","Windows Server","ODX","PowerShell"]
---
Load these PowerShell functions and life is easy :)

```powershell
function Get-ODX
{
    [CmdletBinding()]
    Param()
    $a = (get-ItemProperty hklm:\system\currentcontrolset\control\filesystem -name "FilterSupportedFeaturesMode").filtersupportedfeaturesmode
    $b = (Get-ItemProperty hklm:\system\currentcontrolset\services\FsDepends -Name "SupportedFeatures").SupportedFeatures
    switch ($a)
    {
        0 {"ODX is enabled"}
        1 {"ODX is disabled"}
    }
    switch ($b)
    {
        3 {"Filter driver supports ODX"}
        default {"Filter driver does not support ODX"}
    }
}

function Set-ODX
{
    [CmdletBinding()]
    Param(
        [switch]$Enable,
        [switch]$Disable
    )

    if ($Enable)
    {
        Set-ItemProperty hklm:\system\currentcontrolset\control\filesystem -Name "FilterSupportedFeaturesMode" –Value 0
    }
    elseif ($Disable)
    {
        Set-ItemProperty hklm:\system\currentcontrolset\control\filesystem -Name "FilterSupportedFeaturesMode" –Value 1
    }
    else
    {
        Write-Error "Use Enable or Disable switch!"
    }
}
```