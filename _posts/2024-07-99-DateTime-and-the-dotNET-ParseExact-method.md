# DateTime and using the .NET ParseExact method

As I mention in [ADF-Representation post](_posts\2024-07-10-Atlassian-Doc-Format-ADF-Representation.md) I work with Jira day to day, and while doing so, I process emails to extract details of the notified issue, that I use to create `New-Jira` issues.

I recently shared my `New-Jira` script with a colleague who works in the US, but they had an error when using it that I do not see.

Heres a review of the "what" and "why", hopefully you can find it useful.

### the original PowerShell, worked on an en-IE locale system

```powershell
## Convert MMddYYYYtime as ddMMYYYYtime using regex groups
    1    $MMddYYYYtime = $activityObject.'Created Date' -replace 'Created Date : '
    2    $ddMMYYYYtime = $MMddYYYYtime -replace '(\d{1,2}/)(\d{1,2}/)(.*)' , '$2$1$3'
    3    $activityObject.'Created Date' = Get-Date $ddMMYYYYtime
```

#### $activityObject represents an email object

More specifically, $activityObject is a `PSCustomObject` that represents the text extracted from an email, from the email body. The data is generated in PowerShell using the `Microsoft.Office.Interop.Outlook` assembly to read body text from my email. The email happens to be a new issue notification from our Siebel ticketing system.

$activityObject has a number of properties, one of which is `"Created Date"`.
The initial value of that is something like `Created Date : 7/22/2024 10:49:13 AM`
The value is month / day / year \_ time, which is what lead to my problem.

**At line #1 above:** `-replace 'Created Date : '` is used to trim the descriptive detail leaving a string that represents a date time value. [^1]
**At line #2:** uses -replace with regex groups to reorder the `month/day` portions of the string to appear as `day/month`.
**At line #3:** modifies the 'Created Date' property with the Get-Date function.

With that working on my en-IE local system, we discovered that did not work on their en-US local system. The issue ocurred at line #3, where `Get-Date $ddMMYYYYtime` had error when used at the en-US local system, so I went hunting for a solution.

The solution found is the .NET `[DateTime]::ParseExact()` method.

### My solution

```powershell
## Convert MMddYYYYtime as ddMMYYYYtime using [DateTime]::ParseExact method
$MMddYYYYtime = '7/22/2024 1:49:13 PM'
$template = 'M/d/yyyy h:mm:ss tt'
$activityObj.'SR Created Date' = [DateTime]::ParseExact($MMddYYYYtime, $template, $null)
```

The value "tt" represents the AM/PM designator. [^2] [^3]

[^1]: You could also use `String.Trim()` / `String.TrimStart()` method, but those are slower when examined using `Measure-Command{}`
[^2]: 'tt' is a custom format specifier that represents the AM/PM designator. [More information: ttSpecifier](https://learn.microsoft.com/en-us/dotnet/standard/base-types/custom-date-and-time-format-strings#ttSpecifier)
[^3]: system.datetime.parseexact [More information: parseexact](https://learn.microsoft.com/en-us/dotnet/api/system.datetime.parseexact)
