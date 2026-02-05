# Extracting PowerShell Function Names with the AST

*PowerShell's **Abstract Syntax Tree (AST)** is part of the language, and this post writes about some of its capabilities.
Reference [system.management.automation.language.ast](https://learn.microsoft.com/en-us/dotnet/api/system.management.automation.language.ast)*

As time goes by, we all find better ways of writing old code, such as changing an old script/function for the better.

In this post I look back at an old [Bluesky](https://bsky.app/profile/tecknikp.bsky.social/post/3lhesandezc2f) post covering `Get-FunctionName` - a function from my [#PesterUtility](https://github.com/pauljnav/PesterUtility) repo. And in this, I write about how I extract function names (from a script) using `ScriptBlockAst` and `FunctionDefinitionAst` parsing, and made progress in the methods I used as I learned more, and got a little better at PowerShell.

The method used in the latest version of the function outputs more than just the originals function "Name", and now it outputs many of the useful properties that are to be found on the `FunctionDefinitionAst` class. Reference [FunctionDefinitionAst](https://learn.microsoft.com/en-us/dotnet/api/system.management.automation.language.functiondefinitionast).

The earlier versions work, but were less full of the object goodies that the AST offers us in PowerShell. And with those earlier versions, I hadn't studied the AST so well, meaning that I didn't see the extra capabilities it offered.

So this story also shows me making a little progress with time.

## Get-FunctionName.ps1
**Q:** Why have a `Get-FunctionName` function at all?

**A:** Sometimes you just need a list of function names from your PowerShell scripts without loading/running them. This comes up when:
* Auditing scripts
* Generating documentation
* Linting or static analysis
* Checks on code coverage when testing.

For me, as the [#PesterUtility](https://bsky.app/hashtag/PesterUtility) hashtag suggests, I use it as a utility when [#Pester](https://bsky.app/hashtag/Pester) testing.

I had a need to reliably (and easily) list the function names from scripts, because I wanted to make sure that I had defined a test for every function with no escapees.

---

## Why use the AST?

I knew regex patterns could help to identify `"function Do-Something"` and `"filter Do-SomethingElse"`, but thats re-inventing the wheel because of the AST, and it could prove error-prone.

The AST represents PowerShell code as a structured tree. ANd parsing a script into an AST lets you inspect its contents **without executing anything**.

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

## Some public AST Modules we can reference

[Select-Ast](https://www.powershellgallery.com/packages/Select-Ast) by [Kevin Marquette](https://github.com/KevinMarquette/Select-Ast)

[PSFunctionTools](https://www.powershellgallery.com/packages/PSFunctionTools) by [Jeff Hicks](https://github.com/jdhitsolutions/PSFunctionTools)

---

## V1 Get-FunctionName.ps1

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

I used `::Tokenize()` first, and iterating the output and could see that each token had a `.Type`, a `.Content`, and so I parsed those, extracting tokens whose `.Type` equalled `Keyword` and whose `.Content` equalled `function` or `Filter`. But to be assured the item was a function, I neded an inner loop to check if the next token had `.Type` equalling `CommandArgument`.

This gave its output as a raw name listing, useful, but not objects.
```PowerShell
Get-ThisFunction
Get-ThatFunction
Get-TheOtherFunction
```

### How did it work?

An example is best....

That `Tokenize()` method chunked the script content into many tokens (one 1200 line script yielded 6562 tokens).

Below you can follow the logic checks `$token.Type -eq "Keyword"` with `$token.Content -eq "filter"` that isolates the first token in the example. And also the logic check for `$nextToken.Type -eq "CommandArgument"` too. In this way, the token parsing was able to collect tokens matching those criteria, and read out the Content (the function name) was `New-PrintTextBox`.

And that's how the job was done with version 1.

So much complication in that version! 🤦‍♂️

```PowerShell
# An example of a token matching whose `.Content` equalled `Filter`:
TypeName: System.Management.Automation.PSToken

Content     : filter
Type        : Keyword
Start       : 8762
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

FYI: a simple way to get the token details is:
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

# Collect the first matching token and the token that follows
$tokens | Where-Object Start -ge 8762 | Select-Object -First 2           # manually specify the Start value
$tokens | Where-Object Start -ge $token1.Start | Select-Object -First 2  # supply the Start value from $token1
```

## V2 Get-FunctionName.ps1

Version 2 was significantly better because I had learned a bit more, and I let the power of the PowerShell AST do more. Just by using a different method as follows:

### Version 2 used `[Parser]::ParseFile()` method as follows:

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

This version is a lot smaller and better, it got easier thanks to additional study of the AST with FunctionDefinitionAst.

And with the use of `Select-Object Name` it has improved it's outputs with a good object pattern.

```PowerShell
Name
----
Get-ThisFunction
Get-ThatFunction
Get-TheOtherFunction
```
---

## V3 Get-FunctionName.ps1
The next version came after some fucus on the study of the members/methods available with AST objects.

This version uses the ast.findall() method. Reference [findall](https://learn.microsoft.com/en-us/dotnet/api/system.management.automation.language.ast.findall)

To study the AST in more detail, I saved a set of dummy functions into a `sample.ps1` script as follows:
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

The `sample.ps1` script is not a functional set of code, merely a set of dummy functions that do nothing but exist. But, by loading and studying the resulting AST details, you can open a world of AST knowledge for yourself.

At the heart of the V3 Get-FunctionName.ps1 function is the same ast `[Parser]::ParseFile()` method as before, then replacing `$ast.EndBlock.Statements` / `$ast.EndBlock` with the `$ast.FindAll()` method. See the online reference [ast.findall](https://learn.microsoft.com/en-us/dotnet/api/system.management.automation.language.ast.findall)

The `searchNestedScriptBlocks` Boolean usage also opened a nice little doorway to search nested functions and script block expressions by incorporating as a switch parameter, reference [switch-parameters](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_functions_advanced_parameters?#switch-parameters).

---

## The Goodies

This is the core logic:

```powershell
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
    # Resolve whatever path is given to its LiteralPath
    $resolvedPath = (Resolve-Path -LiteralPath $Path -ErrorAction Stop).ProviderPath

    # ::ParseFile() method parses the scriptblocks from the script into AST object details.
    $ast = [System.Management.Automation.Language.Parser]::ParseFile(
        $resolvedPath, [ref]$token, [ref]$errors
    )

    # Sanity check the object is valid ScriptBlockAst
    if ($ast -isnot [System.Management.Automation.Language.ScriptBlockAst]) { return }

    # Extract FunctionDefinitionAst
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

But, the AST type provides a lot more properties in various depths, so let's introduce caclulated properties and surface extra details.

```PowerShell
    # In addition to Name, output a list of properties, incl. some caclulated properties.
    $FunctionDefinitionAst | Select-Object Name, IsFilter, Parameters,
        @{Name = 'LineNumber'; Expression = { $_.Extent.StartLineNumber } },
        @{Name = 'FilePath';   Expression = { $_.Extent.File } },
        @{Name = 'FileName';   Expression = { [System.IO.Path]::GetFileName($_.Extent.File) } },
        @{Name = 'Text';       Expression = { $_.Extent.Text } }
```

Result:

```PowerShell
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

And we could get more, but this is good enough for my needs..

---

## What's happening here?

### Parsing the file

```powershell
$ast = [System.Management.Automation.Language.Parser]::ParseFile(...)
```

This converts the script into an AST object.
Nothing is executed.

---

### Sanity test of the $ast object - the `return` operator breaks out of the Process{} block, next object (if any) entering the pipeline is processed.

```powershell
if ($ast -isnot [System.Management.Automation.Language.ScriptBlockAst]) { return }
```

If for some reason, the item is not AST, break the loop This filters the tree to function definitions only.
NB: we dont use the "break" operator as that would also terminate the pipeline processing in Process{}

---

### Excluding nested functions

```powershell
$IncludeNestedFunctions = $true # or $false
$FunctionDefinitionAst = $ast.FindAll(
    {
        param($node)
        $node -is [System.Management.Automation.Language.FunctionDefinitionAst]
    }, $IncludeNestedFunctions
)
```

This calls the `.FindAll()` method of the ast class.

The `.FindAll()` method traverses the ast tree structure, meaning we dont need to write any looping logic to check every branch, PowerShell implements the loopingg.

The Predicate `{ param($node) ... }` captures code input to the scriptblock into a variable `$node`, and it allows the block to iterate over every piece of code (node) the parser finds. The `.FindAll()` method does the hard work of walking through every piece of code found (each node) in the tree, and for every node it finds, it places that into the $node variable for the logic test. The var name can be any legitimate variable name, it does not have to be `$node`.

The comparison operator: `... -is [.Language.FunctionDefinitionAst]` This is the specific "type" check, a logic test that allows the scriptblock to grab function/filter FunctionDefinition  and ignore variables, strings, or loops.

---

### Searching the full tree

```PowerShell
FindAll(..., $true) # when $IncludeNestedFunctions -eq $true
#OR
FindAll(..., $false) # when $IncludeNestedFunctions -eq $false
```

`$IncludeNestedFunctions` This boolean tells the search whether to stop at the top level or "go deep" into child script blocks to find functions contained within  patent functions. Here it is used as an input parameter, enabling the caller to make that choice.

`$true` tells PowerShell to **search** nested script blocks, so functions inside `if`, `try`, or other deeper blocks are still found.

`$false` tells PowerShell to **not search** nested script blocks, to return only the "top-level" functions found.

---

```PowerShell
    # In addition to Name, output a list of properties, incl. some caclulated properties.
    $FunctionDefinitionAst | Select-Object Name,
        @{Name = 'LineNumber'; Expression = { $_.Extent.StartLineNumber } }
```

The $FunctionDefinitionAst object has nested properties/members. Use the Help system to see the details:
```PowerShell
$FunctionDefinitionAst | Get-Member
```

One nested property in the example above is: `$FunctionDefinitionAst.Extent.StartLineNumber`.

By using `Expression = { $_.Extent.StartLineNumber }`, we select that value from the object `$_` in the pipeline `$FunctionDefinitionAst`.

---

## Why it  works well

* No code execution or loading
* Fast
* Accurate
* Works on `.ps1` and `.psm1`
* Ideal for tooling and automation

This pattern is a great foundation for more advanced PowerShell analysis—like exporting function metadata and validating module structure.

---

Later, check out:

* Returning full AST objects instead of names
* Detecting exported vs private functions
