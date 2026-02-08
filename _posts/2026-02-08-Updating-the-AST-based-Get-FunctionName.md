# Extracting PowerShell Function Names with the AST

As time goes by, we all find better ways of writing old code, such as changing an old script/function for the better.

In this post I look back at an old [Bluesky](https://bsky.app/profile/tecknikp.bsky.social/post/3lhesandezc2f) post covering `Get-FunctionName` - a function from my [#PesterUtility](https://github.com/pauljnav/PesterUtility) repo. And in this, I write about how I extract function names (from a script) using `ScriptBlockAst` and `FunctionDefinitionAst` parsing, and as I learned more, I made progress in the methods I used, and I got a little better at PowerShell.

*PowerShell's **Abstract Syntax Tree (AST)** is part of the language, and this post covers some of its capabilities.
See Reference [system.management.automation.language.ast](https://learn.microsoft.com/en-us/dotnet/api/system.management.automation.language.ast)*

### Code improvement with time

I write about 3 versions of a function that captures function names from `.ps1`/`.psm1` scripts.

The method used in the latest version outputs more than just function "Name", as it now outputs additional useful properties that are found on the `FunctionDefinitionAst` class as used when testing a function. See reference [FunctionDefinitionAst](https://learn.microsoft.com/en-us/dotnet/api/system.management.automation.language.functiondefinitionast).

The earlier versions work, but were less full of the object goodness that the AST offers us in PowerShell. And when creating those earlier versions, I hadn't studied the AST so well, and I didn't see the capabilities it offered.

So this story also shows me making a little code improvement with time.

## Get-FunctionName

**Q:** Why have a `Get-FunctionName` command at all?

**A:** Sometimes you just need a list of function names from your PowerShell scripts without loading/running them. This comes up when:
* Auditing scripts
* Generating documentation
* Linting or static analysis
* Checks on code coverage when testing.

For me, as the [#PesterUtility](https://bsky.app/hashtag/PesterUtility) hashtag suggests, I use it as a utility when [#Pester](https://bsky.app/hashtag/Pester) testing.

I had a need to reliably (and easily) list the function names from scripts, because I wanted to make sure that I had defined a test for every function with no escapees.

---

## Why use the AST?

[Regex](https://learn.microsoft.com/en-us/dotnet/standard/base-types/regular-expression-language-quick-reference) is great. [Select-String](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/select-string) is also great. We know that regex search patterns could help to identify `"function Do-Something"` and `"filter Do-SomethingElse"`, but that's re-inventing the wheel when the AST exists, and searching with text patterns could prove error-prone.

For example, a function named `"DoSomething"` is also legitimate PowerShell. And through the power of AST, we can represent PowerShell code as a structured tree, as objects with properties. And parsing a script into an AST lets you inspect all such contents **without executing anything**.

PowerShell's **Abstract Syntax Tree (AST)** makes this easy and safe. Reference [ast](https://learn.microsoft.com/en-us/dotnet/api/system.management.automation.language.ast)

Easy and safe means:

* No side effects
* No security risk
* No dependency loading

Perfect for tooling.

---

## The core idea

We:

1. Parse a script file into an AST
2. Walk the AST
3. Select *top-level* function definitions
4. Output their names
5. Add more functionality in V3 - select *child-level* function definitions, etc.

"Top-level" here means **functions that are not defined inside other functions**.

---

As references; many other PowerShellers have the better code published, just check the PowerShell Gallery for [Tags:"ast"](https://www.powershellgallery.com/packages?q=Tags%3A%22ast%22).

## Some public AST Modules for reference

[Select-Ast](https://www.powershellgallery.com/packages/Select-Ast) by [Kevin Marquette](https://github.com/KevinMarquette/Select-Ast) has
`Select-AST -AstScriptBlock $ScriptBlock -Type FunctionDefinitionAst` functionality.

[PSFunctionTools](https://www.powershellgallery.com/packages/PSFunctionTools) by [Jeff Hicks](https://github.com/jdhitsolutions/PSFunctionTools) has a `Get-FunctionName -Path $ScriptPath` functionality.

---

## V1 Get-FunctionName

Originally the function used a process block driven by `[PSParser]::Tokenize()` method as follows:

```PowerShell
process {
    # Read the content of the script
    $scriptContent = Get-Content -Path $ScriptPath -Raw

    # Tokenize the PowerShell script content
    $tokens = [System.Management.Automation.PSParser]::Tokenize($scriptContent, [ref]$null)

    # Iterate through the tokens
    for ($i = 0; $i -lt $tokens.Count; $i++) {
        $token = $tokens[$i]

        # Check if the token represents the start of a function or filter definition
        if ($token.Type -eq "Keyword" -and ($token.Content -eq "function" -or $token.Content -eq "filter")) {
            # Find the corresponding function or filter name
            for ($j = $i + 1; $j -lt $tokens.Count; $j++) {
                $nextToken = $tokens[$j]
                if ($nextToken.Type -eq "CommandArgument") {
                    # PSCustomObject
                    Write-Output (
                        [PSCustomObject] @{
                            FunctionName = $nextToken.Content
                        }
                    )
                    break # Exit the inner loop once the function or filter name is found
                }
            }
        }
    }#for (outer loop)
}
```

The first version used `::Tokenize()`, and when iterating the output, I could see that each token had a `.Type`, a `.Content`, and so I parsed those, extracting tokens whose `.Type` equalled `Keyword` and whose `.Content` equalled `function` or `Filter`. But to be assured the item was a function, I needed an inner loop to check if the next token had `.Type` equalling `CommandArgument`.

This gave its output as a raw name listing, useful, but not objects.
```PowerShell
Get-ThisFunction
Get-ThatFunction
Get-TheOtherFunction
```

### How did it work?

That `Tokenize()` method chunked the script content into many tokens (one 1200-line script yielded 6562 tokens).

Below you can follow the logic checks `$token.Type -eq "Keyword"` with `$token.Content -eq "filter"` that isolates the first token in the example. And also the logic check for `$nextToken.Type -eq "CommandArgument"` too. In this way, the token parsing was able to collect tokens matching those criteria, and read out the Content (the function name) was `New-PrintTextBox`.

So much complication in that version! 🤦‍♂️ But that was how the job was done with version 1.

```PowerShell
# An example of a matching token where `.Content` equalled `filter`:
TypeName: System.Management.Automation.PSToken

Content     : filter
Type        : Keyword
Start       : 8762     # This Start value is used later.
Length      : 6
StartLine   : 176
StartColumn : 1
EndLine     : 176
EndColumn   : 7

# And the token following after:

Content     : New-PrintTextBox
Type        : CommandArgument
Start       : 8769
Length      : 12
StartLine   : 176
StartColumn : 8
EndLine     : 176
EndColumn   : 20
```

FYI: How to view the token details from the script content is:

```PowerShell
# count of lines in script
$scriptContent = Get-Content -Path $scriptPath -Raw
($scriptContent -split '\n' ).count

# count tokens
$tokens | Measure-Object

# Using Get-Member, identify the TypeName
$tokens | Where-Object Content -eq "filter" | Get-Member

# Collect the first matching token into an OutVariable named "token1"
$tokens | Where-Object Content -eq "filter" | Select-Object -First 1 -OutVariable token1
# Above uses `-OutVariable token1` to assign the output to a variable $token1, but it's the same as the following
# $token1 = $tokens | Where-Object Content -eq "filter" | Select-Object -First 1

# Collect the first matching token and the token that follows
$tokens | Where-Object Start -ge 8762 | Select-Object -First 2           # manually specify the Start value
$tokens | Where-Object Start -ge $token1.Start | Select-Object -First 2  # supply the Start value from $token1
```

---

## V2 Get-FunctionName

Version 2 was significantly better because I had learned a bit more, and I let the power of the PowerShell AST do more. Just by using a different method as follows:

Version 2 used `[Parser]::ParseFile()` method as follows:

```PowerShell
Process {
    $token = $null
    $errors = $null

    # ParseFile
    $ast = [System.Management.Automation.Language.Parser]::ParseFile($fileName, [ref]$token, [ref]$errors)

    # From $ast.EndBlock.Statements, extract FunctionDefinitionAst object
    $functionNames = $ast.EndBlock.Statements |
    Where-Object {$_ -is [System.Management.Automation.Language.FunctionDefinitionAst]} |
        Select-Object Name

    # PSCustomObject
    Write-Output $functionNames
}
```

Version 2 is a lot smaller and better, it got easier thanks to additional study of the AST with FunctionDefinitionAst.

And with the use of `.. | Select-Object Name` it has improved its outputs with a good PS object pattern, as that sends the Property named `Name` and the value within to the output.

```PowerShell
Name
----
Get-ThisFunction
Get-ThatFunction
Get-TheOtherFunction
```

---

## V3 Get-FunctionName

The next version came after some more study of the members/methods available with AST objects.

Version 3 uses the `ast.findall()` method. Reference [findall](https://learn.microsoft.com/en-us/dotnet/api/system.management.automation.language.ast.findall)

To study the AST in more detail, save a set of dummy functions into a `sample.ps1` script and used that as a learning aid, as follows:

```PowerShell
### 1. Function with Helper Functions and a Filter
function Get-SystemReport ([string]$param1, [int]$param2) {
    function Get-Timestamp { Get-Date -Format "yyyy-MM-dd HH:mm" }
    function Get-User { $env:USERNAME }
    filter Mark-Urgent { if ($_ -match "Error") { "[URGENT] $_" } else { $_ } }

    $msg = "Report generated by $(Get-User) at $(Get-Timestamp)"
    $msg | Mark-Urgent
}

### 2. Function with Two Filters
function Invoke-DataCleanup {
    filter Remove-Whitespace { $_.Trim() }
    filter Deny-Empty { if (-not [string]::IsNullOrWhiteSpace($_)) { $_ } }

    $rawData = "  item1  ", " ", "  item2  "
    $rawData | Remove-Whitespace | Deny-Empty
}

### 3. Function with 5 Variable Types
function Set-EnvironmentConfig {
    [string]$Name     = "Production"
    [int]$Version     = 3
    [bool]$IsActive   = $true
    [array]$Tags      = @("Web", "Azure", "Secure")
    [hashtable]$Meta  = @{ Owner = "Admin"; ID = 101 }

    Write-Host "Configuring $Name v$Version..."
}
```

The `sample.ps1` script is not functional set of code, merely a set of dummy functions that do nothing but exist. And, by loading and studying the resulting AST details, you can open a world of AST knowledge for yourself.

At the heart of the V3 Get-FunctionName.ps1 function is the same ast `[Parser]::ParseFile()` method as before, then replacing `$ast.EndBlock.Statements` / `$ast.EndBlock` with the `$ast.FindAll()` method.
See the reference [ast.findall](https://learn.microsoft.com/en-us/dotnet/api/system.management.automation.language.ast.findall)

And the `searchNestedScriptBlocks` Boolean usage opened a nice option to search nested functions and script block expressions by incorporating as a switch parameter. See the reference [switch-parameters](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_functions_advanced_parameters?#switch-parameters).

---

## To the good stuff

This is the core logic:

```PowerShell
param (
    [Parameter(
        Mandatory,
        ValueFromPipeline,
        ValueFromPipelineByPropertyName)]
    # Path to script under review
    [string]$Path,

    # optionally include nested functions in search
    [Parameter()]
    [switch]$IncludeNestedFunctions
)

Begin {
    $token = $null
    $errors = $null
}

Process {
    # Resolve the path to its LiteralPath.
    $resolvedPath = (Resolve-Path -LiteralPath $Path -ErrorAction Stop).ProviderPath

    # ::ParseFile() method parses the scriptblocks from the script into AST object details.
    $ast = [System.Management.Automation.Language.Parser]::ParseFile(
        $resolvedPath,
        [ref]$token,
        [ref]$errors
    )

    # Sanity check the object is valid ScriptBlockAst
    if ($ast -isnot [System.Management.Automation.Language.ScriptBlockAst]) { return }

    # Extract FunctionDefinitionAst with $IncludeNestedFunctions true/false
    $FunctionDefinitionAst = $ast.FindAll( {
            param($node)
            $node -is [System.Management.Automation.Language.FunctionDefinitionAst]
        }, $IncludeNestedFunctions
    )
}

End {
}
```

At end of the Process block, we can select the `Name` property, the exact detail needed.
```PowerShell
# Select Name as the output from the function
$FunctionDefinitionAst | Select-Object Name
```

Result:

```PowerShell
Name
----
Get-ThisFunction
Get-ThatFunction
```

And, the AST type provides a lot more properties in various nested levels, so let's surface extra details by *"walking around"* in the detail discovered for the `sample.ps1` script.

---

## What's happening at the key portions?

### Define path to the input file

```PowerShell
$Path = '\path\to\sample.ps1'
$resolvedPath = (Resolve-Path -LiteralPath $Path -ErrorAction Stop).ProviderPath
```

### Parsing the file

```PowerShell
# false when not searching for nested/child functions.
$IncludeNestedFunctions = $false

$token = $null
$errors = $null

$ast = [System.Management.Automation.Language.Parser]::ParseFile( ... )
```

This converts the script into an AST object.
Nothing is executed.

### Sanity test of the $ast object

The `return` operator returns the script flow out from the Process{} block if triggered.

```PowerShell
if ($ast -isnot [System.Management.Automation.Language.ScriptBlockAst]) { return }
```

If for some reason, the item is not AST, return from the loop
NB: we don't use the "break" operator as that would also terminate the pipeline processing in Process{}

---

### Excluding nested functions

```PowerShell
$IncludeNestedFunctions = $false

$FunctionDefinitionAst = $ast.FindAll( {
        param($node)
        $node -is [System.Management.Automation.Language.FunctionDefinitionAst]
    }, $IncludeNestedFunctions
)
```

This calls the `.FindAll()` method of the ast class.

The `.FindAll()` method traverses the ast tree structure, meaning we don't need to write any looping logic to check every branch, PowerShell implements the looping as it searches the AST for all nodes.

When not IncludeNestedFunctions, the logic checks every branch, but does not dive into any nested functions found - all without having to write any flow control.

The Predicate `{ param($node) ... }` captures code input to the scriptblock into a variable. I'm allocating `$node` for that, but the var name can be any legitimate variable name, it does not have to be `$node`.

The `.FindAll()` method does the hard work of walking through every node of code found in the tree, and for every node it finds, it places that into the $node variable for the logic test.

The comparison operator: `... -is [.Language.FunctionDefinitionAst]` This is the specific "type" check, a logic test that allows the scriptblock to grab function/filter FunctionDefinition and to ignore variables, strings, or loops.

The variable $FunctionDefinitionAst now holds those node objects.

### Searching the full tree

```PowerShell
FindAll(..., $true) # when $IncludeNestedFunctions -eq $true
#OR
FindAll(..., $false) # when $IncludeNestedFunctions -eq $false
```

`$IncludeNestedFunctions` This boolean tells the search whether to stop at the top level or "go deep" into child script blocks to find functions contained within parent functions. Here it is used as an input parameter, enabling the caller to make that choice.

`$true` tells PowerShell to **search** nested script blocks, so functions inside `if`, `try`, or other deeper blocks are still found.

`$false` tells PowerShell to **not search** nested script blocks, to return only the "top-level" functions found.

---

### Let's take a closer look

```PowerShell
# Measure the number of Function Definitions detected
$FunctionDefinitionAst | Measure-Object

Count    : 3
```

### Use the Help system to see details:

```PowerShell
# Use the Help system, Get-Member to see definitions on the object
$FunctionDefinitionAst | Get-Member
```
The output from above delivers too much info we don't need, so I don't repeat it here. Using the next variation we simplify the output.

```PowerShell
# Use Get-Member to see only the property definitions on the object
$FunctionDefinitionAst | Get-Member -MemberType Property

TypeName: System.Management.Automation.Language.FunctionDefinitionAst

Name       MemberType Definition
----       ---------- ----------
Body           Property   System.Management.Automation.Language.ScriptBlockAst Body {get;}
Extent         Property   System.Management.Automation.Language.IScriptExtent Extent {get;}
IsFilter       Property   bool IsFilter {get;}
IsWorkflow     Property   bool IsWorkflow {get;}
Name           Property   string Name {get;}
Parameters     Property   System.Collections.ObjectModel.ReadOnlyCollection[System.Management.Automation.Lang...]
Parent         Property   System.Management.Automation.Language.Ast Parent {get;}
```

The Get-Member output gives an object having 7 properties, Name, Body, Extent, IsFilter, Parent, etc.

### Take a look at two - Name and IsFilter by selecting only those.

```PowerShell
$FunctionDefinitionAst | Select-Object Name, IsFilter

Name                  IsFilter
----                  --------
Get-SystemReport         False
Invoke-DataCleanup       False
Set-EnvironmentConfig    False
```

That output yields two basic properties from each of the three objects.

### Go deeper - take a look at nested data under `.Extent`

Let's specify `Select-Object` with the `-ExpandProperty` parameter to see the nested details.

```PowerShell
$FunctionDefinitionAst | Select-Object -First 1 -ExpandProperty Extent

File                : C:\MyDocs\MyGit\repos\PesterUtility\.project\sample.ps1
StartScriptPosition : System.Management.Automation.Language.InternalScriptPosition
EndScriptPosition   : System.Management.Automation.Language.InternalScriptPosition
StartLineNumber     : 2
StartColumnNumber   : 1
EndLineNumber       : 9
EndColumnNumber     : 2
Text                : function Get-SystemReport ([string]$param1, [int]$param2) {
                        function Get-Timestamp { Get-Date -Format "yyyy-MM-dd HH:mm" }
                        function Get-User { $env:USERNAME }
                        filter Mark-Urgent { if ($_ -match "Error") { "[URGENT] $_" } else { $_ } }

                        $msg = "Report generated by $(Get-User) at $(Get-Timestamp)"
                        $msg | Mark-Urgent
                    }
StartOffset         : 52
EndOffset           : 396
```

Lots of great information from the resulting object:

File, StartLineNumber, EndLineNumber, Text, and more. And the property data could easily be surfaced, and that's what v4 does.

One nested property in the example above is: `$FunctionDefinitionAst.Extent.StartLineNumber`. Let's select StartLineNumber from each object.

```PowerShell
$FunctionDefinitionAst | ForEach-Object Extent | Select-Object StartLineNumber

StartLineNumber
---------------
              2
             12
             21
```

As you can see, selecting data from the nested structure of the object is possible, but that basic method is unsuitable when you want to output details from many different levels of the object.

### A look at calculated properties

At this point let's introduce `calculated properties` to cope with the nested depth of the tree structure of the object. Calculated properties prove useful to surface some of that data to the user, and we can use this in the functions output.

See reference [about_calculated_properties](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_calculated_properties)

Taking a look at the `Name` property, and one of the deeper properties `.Extent.StartLineNumber` surfaced as a calculated property.

```PowerShell
$FunctionDefinitionAst | Select-Object Name,
    @{Name = 'LineNumber'; Expression = { $_.Extent.StartLineNumber } }

Name                  LineNumber
----                  ----------
Get-SystemReport               2
Invoke-DataCleanup            12
Set-EnvironmentConfig         21
```

By using `Expression = { $_.Extent.StartLineNumber }`, we select that value from the objects deeper layer by referencing the object `$_` from the pipeline, and we give it a new name `LineNumber`.

### Put this together and capture more properties

Selecting the top-level properties Name, IsFilter, Parameters.
Also select properties nested under the .Extent property; StartLineNumber, File, Text.

```PowerShell
$FunctionDefinitionAst | Select-Object Name, IsFilter, Parameters,
    @{Name = 'LineNumber'; Expression = { $_.Extent.StartLineNumber } },
    @{Name = 'FilePath';   Expression = { $_.Extent.File } },
    @{Name = 'FileName';   Expression = { Split-Path -Leaf $_.Extent.File } },
    @{Name = 'Text';       Expression = { $_.Extent.Text } }
```

### Taking one to illustrate some concepts of calculated properties.

The property named `FileName` illustrates the object in the pipeline `$_`, and the expression uses `$_.Extent.File` to capture the `File` property nested under `$_.Extent`. That value is given to the `Split-Path -Leaf` command to derive the leafname of that file. Leafname being the base filename excluding all path\to\file detail.

And by using calculated properties we can give that data a new name. I choose `Name = 'LineNumber'` in preference over `StartLineNumber`, but you can retain it if you prefer; by using `Name = 'StartLineNumber'` instead (or choose something different).

The combined result of selecting normal and calculated properties generates the appearance of having all those properties on the `$FunctionDefinitionAst` object in a simple and repeatable way.

```PowerShell
# Result:
Name       : Get-ThisFunction
IsFilter   : False
Parameters :
LineNumber : 1
FilePath   : S:\myFunctions\Get-ThisFunction.ps1
FileName   : Get-ThisFunction.ps1
Text       : function Get-ThisFunction {
                [CmdletBinding()]
                [Alias('this')]
                param(
                    [string]$UserName = "Paul Naughton",
                    [switch]$Force
                )

                Begin {
                    # helper function
                    filter Test-Repository {
                        # rest of the function code
```

There are more properties available to expose, but this much is good enough for my needs.

---

This pattern is a great foundation for more advanced PowerShell analysis—like exporting function metadata and validating module structure.

## Why AST works well for Get-FunctionName

* No execution of the function code
* Fast
* Accurate
* Works on `.ps1` and `.psm1`
* Ideal for tooling and automation

---

### Community questions

At this point in the story, I was at a point where I would publish this, but a few on-point questions from the PowerShell community made me realise I had left a few stones unturned. And the benefit of this is that I can now plan another version of `Get-FunctionName`, one that is improved by tackling those questions.

So, thanks to [mdgrs](https://github.com/mdgrs-mei) for reviewing the draft blog, and for bringing the questions.

### What happens if a parsed script has dot sourced files?

A: The AST only includes code from the file you parse; functions in dot-sourced files are not listed unless you parse those files separately.

### What happens if the syntax of a parsed file is broken?

A: If a file has syntax errors, `ParseFile()` still returns an AST plus an error list in `$errors`, so you can report (or take other action) based on that.

Later, check out:

- What kind of suitable errors feature can be added to the existing function?
- Then improve the function.

---

In conclusion, I hope the details above help you to walk your own way around the FunctionDefinitionAst object and its nested details to surface other properties. And generating a dummy set of functions for testing proved a useful method that aids that process.

Also, reach out to see if anyone from the PowerShell community would like to review what you're doing; feedback and questions from mdgrs was ideal - thanks again to **mdgrs**.

The function Get-FunctionName can be found in my GitHub under [PesterUtility](https://github.com/pauljnav/PesterUtility).
