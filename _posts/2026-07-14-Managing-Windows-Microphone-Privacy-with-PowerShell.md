---
layout: post
title: "Managing Windows Microphone Privacy with PowerShell - Part 2"
date: 2026-07-14
categories: [microphone, privacy, powershell]
tags: [microphone, privacy]
author: Paul Naughton
description: "Four helper functions to make global microphone privacy management on Windows faster and safer from the command line."
---

I've added four helper functions to make global microphone privacy management on Windows faster and safer from the command line:

[Get-MicrophoneAccess.ps1](#get-microphoneaccessps1)  
[Set-MicrophoneAccess.ps1](#set-microphoneaccessps1)  
[Enable-MicrophoneAccess.ps1](#enable-microphoneaccessps1)  
[Disable-MicrophoneAccess.ps1](#disable-microphoneaccessps1)  

## Why these functions?

I needed to disable microphone access, and I wanted to use PowerShell (of course).

And, I received great community feedback from none other than [Jeff Hicks](https://github.com/jdhitsolutions/) and [James Brundage](https://github.com/StartAutomating/)

Many thanks to such super contributors!

## 1) Get-MicrophoneAccess.ps1

This function retrieves the Windows microphone consent setting from the registry, and returns a simple object with:

- whether access is enabled
- the current registry value
- the registry path used

It is read-only and safe to run anytime.

Example usage:
```ps1
Get-MicrophoneAccess
```

## 2) Set-MicrophoneAccess.ps1

This function toggles global microphone access using Windows system tooling. It supports two clear modes:

- enable microphone access
- disable microphone access

It also supports PowerShell safety features like WhatIf and Confirm, so you can preview or confirm changes before they happen.

Example usage:
```ps1
Set-MicrophoneAccess -Enable
Set-MicrophoneAccess -Disable
Set-MicrophoneAccess -Disable -WhatIf
Set-MicrophoneAccess -Disable -Force
```

## 3) Enable-MicrophoneAccess.ps1

This function calls Set-MicrophoneAccess to enable global microphone access - a simple wrapper.

Example usage:
```ps1
Enable-MicrophoneAccess
```

## 4) Disable-MicrophoneAccess.ps1

This function calls Set-MicrophoneAccess to disable global microphone access - a simple wrapper.

Example usage:
```ps1
Disable-MicrophoneAccess
```

## Practical workflow

A simple pattern is:

1. Check current state
2. Apply the change
3. Re-check state

That gives immediate verification and keeps behavior predictable in scripts.

Together they provide a clean get/enable/disable collection for managing one specific privacy control in a script-friendly way.

## Next

Naturally, I will expand with additional privacy functions to cover location, webcam, presence, microphone, radios.

# The function goodies

So that the article is shorter for you dear reader, I dont include the function help in the code samples below.

But you can head to my [PSfunctions](https://github.com/pauljnav/PSfunctions) repo in GitHub to download the code under MIT License.

### Get-MicrophoneAccess.ps1

```ps1
function Get-MicrophoneAccess {

<#
.SYNOPSIS
Gets the global microphone access status.

.DESCRIPTION
Retrieves the Windows microphone consent setting from the registry.

.NOTES
Author: Paul Naughton
#>

    [CmdletBinding()]

    param()

    if ($env:OS -notmatch '^Windows') {
        throw "Get-MicrophoneAccess is supported only on Windows."
    }

    $Path = 'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\CapabilityAccessManager\ConsentStore\microphone'

    if (-not (Test-Path $Path)) {
        throw "Registry path not found: $Path"
    }

    $Value = (Get-ItemProperty -Path $Path -Name Value).Value

    [PSCustomObject]@{
        Enabled       = ($Value -eq 'Allow')
        RegistryValue = $Value
        RegistryPath  = $Path
    }
}
```

### Set-MicrophoneAccess.ps1

```ps1
function Set-MicrophoneAccess {

    <#
    .SYNOPSIS
    Enables or disables global microphone access.

    .DESCRIPTION
    Uses the Windows SystemSettingsAdminFlows executable to change the global microphone privacy setting.

    .NOTES
    Author: Paul Naughton
    #>

    [CmdletBinding(
        SupportsShouldProcess = $true,
        ConfirmImpact = 'High',
        DefaultParameterSetName = 'Enable'
    )]

    param(
        [Parameter(Mandatory, ParameterSetName = 'Enable')]
        [switch]$Enable,

        [Parameter(Mandatory, ParameterSetName = 'Disable')]
        [switch]$Disable,

        [switch]$PassThru,

        [switch]$Force
    )

    begin {

        if ($env:OS -notmatch '^Windows') {
            throw "Set-MicrophoneAccess is supported only on Windows."
        }

        $CommandInfo = Get-Command SystemSettingsAdminFlows.exe -ErrorAction SilentlyContinue

        if (-not $CommandInfo) {
            throw "SystemSettingsAdminFlows.exe was not found."
        }

        $Value = if ($Enable) { 1 } else { 0 }
        $Action = if ($Enable) { 'Enable' } else { 'Disable' }

        # If Confirm is explicitly provided, honor it; otherwise Force suppresses the
        # default ShouldProcess confirmation in function scope.
        if ($Force.IsPresent -and -not $PSBoundParameters.ContainsKey('Confirm')) {
            $ConfirmPreference = 'None'
        }
    }

    process {

        if ($PSCmdlet.ShouldProcess('Global Microphone Access', $Action)) {

            SystemSettingsAdminFlows.exe SetCamSystemGlobal microphone $Value

            # check $? execution status of the last command
            if ($? -ne $true) {
                throw "SystemSettingsAdminFlows.exe failed with exit code $LASTEXITCODE."
            }

            if ($PassThru.IsPresent) {
                Start-Sleep -Milliseconds 400
                Get-MicrophoneAccess
            }
        }
    }
}
```

### Enable-MicrophoneAccess.ps1

```ps1
function Enable-MicrophoneAccess {
    <#
    .SYNOPSIS
    Enables global microphone access.

    .DESCRIPTION
    Calls Set-MicrophoneAccess to enable global microphone access.

    .NOTES
    Author: Paul Naughton
    #>

    [CmdletBinding()]
    param(
        [switch]$PassThru,
        [switch]$Force
    )

    Set-MicrophoneAccess -Enable -PassThru:$PassThru -Force:$Force
}
```

### Disable-MicrophoneAccess.ps1

```ps1
function Disable-MicrophoneAccess {
    <#
    .SYNOPSIS
    Disables global microphone access.

    .DESCRIPTION
    Calls Set-MicrophoneAccess to disable global microphone access.

    .NOTES
    Author: Paul Naughton
    #>

    [CmdletBinding()]
    param(
        [switch]$PassThru,
        [switch]$Force
    )

    Set-MicrophoneAccess -Disable -PassThru:$PassThru -Force:$Force
}
```
