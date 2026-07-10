---
layout: post
title: "Managing Windows Microphone Privacy with PowerShell"
date: 2026-07-10
categories: [Microphone, Privacy, powershell]
tags: [Microphone, Privacy]
author: Paul Naughton
description: "Two helper functions to make global microphone privacy management on Windows faster and safer from the command line."
---

# Managing Windows Microphone Privacy with PowerShell

I've added two helper functions to make global microphone privacy management on Windows faster and safer from the command line:

- Get-MicrophoneAccess.ps1
- Set-MicrophoneAccess.ps1

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