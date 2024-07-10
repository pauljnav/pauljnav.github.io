# Jira

## Atlassian Document Format (ADF)

JSON representation
I work with Jira day to day and the company has migrated to the Atlassian cloud, my PowerShell day to day scripts need updating from their on-prem versions, and while working on that project, I found what was working ok for on-prem, was failing when I tried to create a new issue in the cloud.
Turns out, the issue was my code not being aware of the Atlassian Document Format

## Well what is it and what is is used for?

According to https://developer.atlassian.com/cloud/jira/platform/apis/document/structure/
The Atlassian Document Format (ADF) represents rich text stored in Atlassian products.
ADF is a JSON-based format for representing rich text documents in Atlassian applications.
It allows for a structured representation of content, including text, paragraphs, lists,
tables, and other rich text elements. The format is used by Atlassian products such as Jira
and Confluence to store and render rich text content. The generated ADF JSON structure will include the version, document type, and content with
a paragraph type containing the provided text.

### For more information on ADF, see the Atlassian documentation:

https://developer.atlassian.com/cloud/jira/platform/apis/document/structure/

### Thanks to Atlassian for giving us a playground

https://developer.atlassian.com/cloud/jira/platform/apis/document/playground/

### Now for the goodies, a cmdlet you can all use. Hope you find it useful.

Its as simple as they can be, you give the function some text, and it writes a json document back.
Assign its output to a variable, and you can use that when you invoke a REST method to do its thing in your Jira clour instance.
I use it to create Jira from my outlook email, and that might a story for another day.

Maybe my function should be named : `ConvertTo-AtlassianADF`

```powershell
function Get-AtlassianADFjson {
<#
.SYNOPSIS
Convert a text string to an Atlassian Document Format (ADF) JSON representation.

.DESCRIPTION
This function takes a user-provided text string and converts it into a JSON representation
following the Atlassian Document Format (ADF) structure.

.PARAMETER inputText
The text string to be converted to ADF JSON format.

.EXAMPLE
The example converts the string "User entered text" to its corresponding ADF JSON representation
Get-AtlassianADFjson -inputText "User entered text"

.EXAMPLE
The example converts the string "User entered text" to its corresponding ADF JSON representation
Get-AtlassianADFjson "User entered text"

.NOTES
The ADF JSON structure generated will include the version, document type, and content with
a paragraph type containing the provided text.

For more information on ADF, see the Atlassian documentation:
https://developer.atlassian.com/cloud/jira/platform/apis/document/structure/
https://developer.atlassian.com/cloud/jira/platform/apis/document/playground/

#>
    param (
        [Parameter(Mandatory, Position=0)]
        [string]$inputText
    )

    $adf = @{
        version = 1
        type = "doc"
        content = @(
            @{
                type = "paragraph"
                content = @(
                    @{
                        type = "text"
                        text = $inputText
                    }
                )
            }
        )
    }

    $adf | ConvertTo-Json -Depth 10
}
```

```
Convert a text string to an Atlassian Document Format (ADF) JSON representation.


    -------------------------- EXAMPLE 1 --------------------------

    PS C:\>The example converts the string "User entered text" to its corresponding ADF JSON representation

    Get-AtlassianADFjson -inputText "User entered text"




    -------------------------- EXAMPLE 2 --------------------------

    PS C:\>The example converts the string "User entered text" to its corresponding ADF JSON representation

    Get-AtlassianADFjson "User entered text"
```
