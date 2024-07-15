# Jira

## Atlassian Document Format (ADF)

I work with Jira day to day and we have migrated Jira to the Atlassian cloud platform. While working on that, I discovered parts that worked for on-prem didnt work for cloud. One of those was the creation of a new jira issue using the `/issue` endpont, and it turns out that was failing was because my code was defined (by accident) with "latest" as the rest api version. And with api version 3 (the beta) being the latest, my niw issue request failed because it was not using the Atlassian Document Format (ADF) for certain text fields. I have adjusted my scripts to specify version 2, and I they dont need to use ADF, but ADF is the interesting topic for this blog.

## What is AFD and what is is used for?

According to https://developer.atlassian.com/cloud/jira/platform/apis/document/structure/
The Atlassian Document Format (ADF) represents rich text stored in Atlassian products. ADF is a JSON-based format for representing rich text documents in Atlassian applications.
It allows for a structured representation of content, including text, paragraphs, lists, tables, and other rich text elements. The format is used by Atlassian products such as Jira
and Confluence to store and render rich text content. The generated ADF JSON structure will include the version, document type, and content with a paragraph type containing the provided text.

Version 3 of the Jira Cloud platform REST API provides support for the Atlassian Document Format (ADF), and version 3 is the latest version but is in beta. While version 2 is the released version, which is not supporting Atlassian Document Format.
The example below was used with REST version 3 ("latest") successfully so maybe it can help you.

### For more information on ADF, see the Atlassian documentation:

https://developer.atlassian.com/cloud/jira/platform/apis/document/structure/

### Thanks to Atlassian for giving us a playground

https://developer.atlassian.com/cloud/jira/platform/apis/document/playground/

### Now for the goodies, a cmdlet you can all use. Hope you find it useful.

This function is as simple as a function can be. You provide the function with some text, and it outputs a json string back.
Assign that string to a variable, and use that when you invoke the /issue REST method to do its thing in your Jira clour instance.
I use it to create Jira from my outlook email, and that might a story for another day.

Maybe my function should be named : `ConvertTo-AtlassianADF`

```powershell
function Get-AtlassianADFjson {
<#
.SYNOPSIS
Convert a text string to an Atlassian Document Format (ADF) JSON representation.

.DESCRIPTION
This function takes a user-provided text string and converts it into a JSON representation
following the Atlassian Document Format (ADF) structure. Supports one string as its a simple function.

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
