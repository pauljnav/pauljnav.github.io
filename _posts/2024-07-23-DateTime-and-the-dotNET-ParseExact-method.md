# Processing DateTime using the .NET `ParseExact` Method

As mentioned in my [ADF-Representation post](_posts\2024-07-10-Atlassian-Doc-Format-ADF-Representation.md), I work with Jira daily. In doing so, I often process emails to extract details for creating `New-Jira` issues.

Recently, I shared my `New-Jira` script with a colleague in the US, who encountered an error that I didn't have. Here's a review of the issue and the solution I found, which might be useful for you.

### The Original PowerShell Script

```powershell
# Convert MMddYYYYtime to ddMMYYYYtime using regex groups
$MMddYYYYtime = $activityObject.'Created Date' -replace 'Created Date : '
$ddMMYYYYtime = $MMddYYYYtime -replace '(\d{1,2}/)(\d{1,2}/)(.*)', '$2$1$3'
$activityObject.'Created Date' = Get-Date $ddMMYYYYtime
```

#### Explanation

`$activityObject` represents an email object, specifically a `PSCustomObject` that holds text extracted from an email body. This data is generated in PowerShell using the `Microsoft.Office.Interop.Outlook` assembly to read body text from my email, which is a new issue notification from our Siebel ticketing system.

One of the properties of `$activityObject` is `"Created Date"`, with a value like `Created Date : 7/22/2024 10:49:13 AM`. This format (month/day/year time) caused the problem with my original script.

**Line #1**: Uses `-replace 'Created Date : '` to trim the descriptive detail, leaving a string representing a DateTime value. [^1]

**Line #2**: Uses `-replace` with regex groups to reorder the `month/day` portions of the string to appear as `day/month`.

**Line #3**: Updates the 'Created Date' property with the `Get-Date` function.

While this worked on my en-IE locale system, it failed on an en-US locale system at line #3. The `Get-Date $ddMMYYYYtime` command caused an error in the en-US locale system, so I went hunting for a solution. The solution found is the .NET `[DateTime]::ParseExact()` method.

### Now for the goodies, the solution. Hope you find it useful.

```powershell
# Convert MMddYYYYtime to DateTime using [DateTime]::ParseExact method
$MMddYYYYtime = '7/22/2024 1:49:13 PM'
$template = 'M/d/yyyy h:mm:ss tt'
$activityObject.'Created Date' = [DateTime]::ParseExact($MMddYYYYtime, $template, $null)
```

The "tt" in the format string represents the AM/PM designator. [^2]

### Conclusion

Using the .NET `[DateTime]::ParseExact()` method provides a reliable way to parse DateTime strings across different locale settings. [^3]

---

### Footnotes

[^1]: You could also use the `String.Trim()` or `String.TrimStart()` methods, but these are slower when measured using `Measure-Command{}`.
[^2]: 'tt' is a custom format specifier that represents the AM/PM designator. [More information: ttSpecifier](https://learn.microsoft.com/en-us/dotnet/standard/base-types/custom-date-and-time-format-strings#ttSpecifier)
[^3]: More information on `DateTime.ParseExact` can be found here: [parseexact](https://learn.microsoft.com/en-us/dotnet/api/system.datetime.parseexact)
