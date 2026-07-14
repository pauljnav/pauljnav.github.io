---
layout: post
title: "Managing Windows Microphone Privacy with PowerShell - Part 1"
date: 2026-07-10
categories: [microphone, privacy, powershell]
tags: [microphone, privacy]
author: Paul Naughton
description: "Two helper functions to make global microphone privacy management on Windows faster and safer from the command line."
---

I've added two helper functions to make global microphone privacy management on Windows faster and safer from the command line:

[Get-MicrophoneAccess.ps1](#1-get-microphoneaccessps1)  
[Set-MicrophoneAccess.ps1](#2-set-microphoneaccessps1)  

## Why these functions?

I just needed to disable microphone access, and I wanted to use PowerShell (of course).

## 1) Get-MicrophoneAccess.ps1

This function reads the Windows microphone consent setting from the registry and returns a simple object with:

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
```

## Practical workflow

A simple pattern is:

1. Check current state
2. Apply the change
3. Re-check state

That gives immediate verification and keeps behavior predictable in scripts.

Together they provide a clean read/write pair for managing one specific privacy control in a script-friendly way.

## Next

Naturally, I will expand with better privacy functions to cover location, webcam, presence, microphone, radios.

# The function goodies

### Get-MicrophoneAccess.ps1

```ps1

function Get-MicrophoneAccess {

<#
.SYNOPSIS
Gets the global microphone access status.

.DESCRIPTION
Retrieves the current Windows global microphone privacy setting.

This function reports whether global microphone access is enabled or
disabled. It does not modify the current microphone privacy setting.

.EXAMPLE
Get-MicrophoneAccess

Returns the current global microphone access status.

.EXAMPLE
Get-MicrophoneAccess | Format-List

Displays detailed microphone access information.

.NOTES
Author: Paul Naughton
#>

    [CmdletBinding()]

    param()

    $Path = 'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\CapabilityAccessManager\ConsentStore\microphone'

    if (-not (Test-Path $Path)) {
        throw "Registry path not found: $Path"
    }

    $Value = (Get-ItemProperty -Path $Path -Name Value).Value

    [PSCustomObject]@{
        Enabled      = ($Value -eq 'Allow')
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
    Uses the Windows SystemSettingsAdminFlows executable to change the
    global microphone privacy setting.

    .PARAMETER Enable
    Enables global microphone access.

    .PARAMETER Disable
    Disables global microphone access.

    .EXAMPLE
    Set-MicrophoneAccess -Enable

    .EXAMPLE
    Set-MicrophoneAccess -Disable

    .EXAMPLE
    Set-MicrophoneAccess -Disable -WhatIf

    Shows the change that would be performed without modifying the current
    microphone privacy setting.

    .NOTES
    Author: Paul Naughton
    #>

    [CmdletBinding(
        SupportsShouldProcess = $true,
        ConfirmImpact = 'High',
        DefaultParameterSetName = 'Enable'
    )]

    param(

        [Parameter(Mandatory,ParameterSetName = 'Enable')]
        [switch]$Enable,

        [Parameter(Mandatory,ParameterSetName = 'Disable')]
        [switch] $Disable
    )

    begin {

        $CommandInfo = Get-Command SystemSettingsAdminFlows.exe -ErrorAction SilentlyContinue

        if (-not $CommandInfo) {

            $CommandPath = Join-Path $env:windir 'System32\SystemSettingsAdminFlows.exe'

            if (Test-Path $CommandPath) {
                $CommandInfo = Get-Command $CommandPath
            }
        }

        if (-not $CommandInfo) {
            throw "SystemSettingsAdminFlows.exe was not found."
        }

        $Executable = $Command.Source

        $Value  = if ($Enable) { 1 } else { 0 }
        $Action = if ($Enable) { 'Enable' } else { 'Disable' }
    }

    process {

        if ($PSCmdlet.ShouldProcess('Global Microphone Access', $Action)) {

            & $Executable SetCamSystemGlobal microphone $Value

            # check $? execution status of the last command
            if ($? -ne $true) {
                throw "SystemSettingsAdminFlows.exe failed with exit code $LASTEXITCODE."
            }

            Get-MicrophoneAccess
        }
    }
}

```
