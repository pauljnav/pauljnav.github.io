## Quick end easy function Set-QuickAccessItem

I'm changing my pc due to BSOD issues, and wanted a quick way to capture my Quick Access pins at Windows 11.

The result is a simple [PowerShell](https://learn.microsoft.com/en-us/powershell/) function `Set-QuickAccessItem` available at [this gist](https://gist.github.com/pauljnav/61ab057d2ac50dbd87c0d829936c2b8f)

Below is the try/catch blocks, see [the gist](https://gist.github.com/pauljnav/61ab057d2ac50dbd87c0d829936c2b8f) for the functions code.

Inspired by A [Dr. Scripto Blog](https://devblogs.microsoft.com/scripting/use-powershell-to-work-with-windows-explorer/)

```powershell

    try {
        $shell = New-Object -ComObject shell.application
        $quickAccessGUID = 'shell:::{679f85cb-0220-4080-b29b-5540cc05aab6}'
        $quickAccessItems = $shell.Namespace($quickAccessGUID).Items()
        $resolvedPath = (Get-Item $Path).FullName

        $isPinned = $quickAccessItems | Where-Object { $_.Path -eq $resolvedPath }

        if ($PSBoundParameters.ContainsKey('Unpin') ) {
            if ($isPinned) {
                # unpin
                $shell.Namespace($Path).Self.InvokeVerb("pintohome")
                Write-Verbose "Unpinned from Quick Access"
            }
        } else {
            if (-not $isPinned) {
                # pin
                $shell.Namespace($Path).Self.InvokeVerb("pintohome")
                Write-Verbose "Pinned to Quick Access"
            }
        }
    } catch {
        Write-Verbose "Error while modifying Quick Access: $_"
    }
```
