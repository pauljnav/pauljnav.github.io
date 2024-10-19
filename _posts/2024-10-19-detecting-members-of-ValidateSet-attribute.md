# Programmatically detecting the members of a ValidateSet attribute within your function.

Well, I struggled with this, so I thought it worth adding to the blog.

I've been developing a reporting module, and I faced a challenge when using `ValidateSet()`. I needed to define a function with 17 options for a parameter, leveraging ValidateSet() for tab completion and easy default selections. And the selection of all 17 options is the default behaviour needed for the reporting application that I was developing for.

I began by defining the function parameter using '*' as a literal wildcard option that would implement the selection of all 17 valid options, but I realised a better design: if no option is selected, the function defaults to all 17 options for that parameter.

This design allows for the flexibility needed for the reporting application. In this way, the user can select option1, or option1 & option2 etc. As the user needed the capability to select any number and combination of options from the list of 17.

### Now for the goodies, a one-liner you can use. Hope you find it useful.
```PowerShell
# Detect ValidateSet attribute members on the 'Status' parameter within the function
$PSCmdlet.MyInvocation.MyCommand.Parameters['Status'].Attributes.ValidValues
```
*Note, the one-liner works on this occasion only because there are no other attribute types defined on it's 'Status' parameter.

### This small script is useful to help explain what I mean.

```Powershell
function Get-Color {
    [CmdletBinding()]
    param (
        [Parameter(Mandatory)]
        [ValidateSet('Red', 'Green', 'Blue')]
        [string]$Color
    )

    # Detect ValidateSet members for the 'Color' parameter within the function
    $parameterName = 'Color'
    $parameter = $PSCmdlet.MyInvocation.MyCommand.Parameters[$parameterName]
    $validateSetAttribute = $parameter.Attributes | Where-Object { $_ -is [System.Management.Automation.ValidateSetAttribute] }

    # Print the ValidValues from  the ValidateSet
    Write-Host "The valid options for the -Color parameter are: $($validateSetAttribute.ValidValues -join ', ')"
}

# Call the function to see the output
Get-Color -Color Green
```

The interesting part is:
`$parameter.Attributes | Where-Object { $_ -is [System.Management.Automation.ValidateSetAttribute] }` 

If you inspected the content of `$parameter.Attributes`, you would get the following output:
```
IgnoreCase ValidValues        TypeId                                                     
---------- -----------        ------                                                     
      True {Red, Green, Blue} System.Management.Automation.ValidateSetAttribute
                              System.Management.Automation.ParameterAttribute
                              System.Management.Automation.ArgumentTypeConverterAttribute
```
That output shows no values having the ParameterAttribute or ArgumentTypeConverterAttribute types, because none such option is defined in the function.
When you develop your own functions, you might be adding those options, so you would need to filter using the `Where-Object` command.

That is why my one-liner does not go to the same effort as that performed by the `Where-Object` example from `function Get-Color`.

Lastly, you could include the `$_.TypeId` in the statement, sboth of these are equivalent, however the first is best.
```PowerShell
$parameter.Attributes | Where-Object { $_ -is [System.Management.Automation.ValidateSetAttribute] }
$parameter.Attributes | Where-Object { $_.TypeId -eq [System.Management.Automation.ValidateSetAttribute] }
```
