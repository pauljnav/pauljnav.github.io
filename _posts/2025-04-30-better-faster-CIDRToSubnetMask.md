# A better and faster CIDR to subnet mask in PowerShell

This long-overdue follow-up to the original post [Convert CIDR to Subnet Mask]({% post_url 2025-01-08-blog-Convert-CIDRToSubnetMask %}), comes with a familiar story.

As coders, we often leave unfinished, rough scripts, quick hacks, or ideas left undone. Later, when we revisit our old work, we spot the rough edges and realise there's room for improvement. This post is part of that process: reflection, refinement, and sharing something better the second time around.

This is one of those, born of a block of code that has been quietly nagging at me ever since I wrote it. Back in January, I needed to convert CIDR to subnet mask for scripting an Azure VM build, and I took the complex route. It was fun to dive into the math and explain how it all worked, but honestly? There was no need.

Maybe this has never happened to you, but I had [forgotten](https://www.merriam-webster.com/dictionary/forget) a better (and much simpler) version that I had made long ago. That forgotten 'gem' is what’s been nagging at me ever since.

Take a peek at the [original]({% post_url 2025-01-08-blog-Convert-CIDRToSubnetMask %}) where I walk through the power-of-two math behind CIDR conversion, including bitwise techniques like `shl` (shift-left) to derive the [subnet mask](https://en.wikipedia.org/wiki/Subnet).

This was my earlier function, using [PowerShell Filter syntax](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_functions#filter-syntax) to provide for simple pipelining on each object entering the pipeline.
```powershell
filter ConvertFromCIDR
{
    Write-Verbose "[ConvertFromCIDR] Enter."
    [string]$bin = [string]1 * $PSItem
    $bin = $bin.PadRight(32,'0')
    $octs = @( $bin.Substring(0,8), $bin.Substring(8,8), $bin.Substring(16,8), $bin.Substring(24,8) )
    $decs = $octs | ForEach-Object { [Convert]::ToByte($_, 2) }
    $decs -join '.'
}
```

This filter is straightforward to follow. Here's how it works:

1. **Input**: Given a number (e.g., 3), the filter creates a string of that many `1`s (e.g., `111`).
2. **Padding**: The `[string]PadRight()` method is then used to pad the string to 32 characters in length, filling the remaining space with `0`s. For example, `111` becomes `11100000000000000000000000000000`.
3. **Splitting**: The string is split into 4 octets (8 bits each) using the `Substring()` method four times.
4. **Conversion**: Each octet in binary, is converted into its decimal equivalent using the `[Convert]::ToByte()` method.
5. **Output**: The decimal values are then joined together with a dot (`.`) to form the familiar dotted-decimal subnet mask format.

### Talk to the LLM
I ask my friendly LLM to turn this into an improved function named "Convert-CIDRToSubnetMask"
```powershell
function Convert-CIDRToSubnetMask {
    param (
        [Parameter(Mandatory)]
        [ValidateRange(0,32)]
        [int]$CIDR
    )
    $bin = ('1' * $CIDR).PadRight(32, '0')
    $octets = $bin -split '(.{8})' -match '\d'
    $subnetMask = ($octets | ForEach-Object { [convert]::ToInt32($_, 2) }) -join '.'
    $subnetMask
}
```
The LLM did good. And I love the new split method `$bin -split '(.{8})'` it has added, that's way better than the `.Substring(0,8)` method 4x times!
Try it, `'11100000000000000000000000000000' -split '(.{8})'` and you can see why I then added the `-match '\d'` (regex numeric match) to remove blank lines.

Another goodie, the LLM also modified the conversion method from `[convert]::ToByte()` to `[convert]::ToInt32()`.

One flaw, as the LLM offering gives no support for support pipeline input, easily fixed by adding [ValueFromPipeline argument](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_functions_advanced_parameters#valuefrompipeline-argument): `[Parameter(Mandatory,ValueFromPipeline)]`

Better [Prompt Engineering](https://en.wikipedia.org/wiki/Prompt_engineering) would have helped too.

## Testing code
So how about testing the function and observing for performance?

### Unit Tests (Pester)
Pester is a testing and mocking framework for PowerShell. See [Pester Quick-start](https://pester.dev/docs/quick-start) and 
[Pester on GitHub](https://github.com/pester/Pester)

*If you dont know about Pester, your going to love getting to know more about it.*

Here's a test suite in Pester v5 to validate the Convert-CIDRToSubnetMask implementation:
```powershell
# Pester v5 Test Script for Convert-CIDRToSubnetMask
Describe "Convert-CIDRToSubnetMask Tests" {

    It "Should return 255.255.255.0 for CIDR 24" {
        Convert-CIDRToSubnetMask -CIDR 24 | Should -Be "255.255.255.0"
    }

    It "Should return 255.255.255.252 for CIDR 30" {
        Convert-CIDRToSubnetMask -CIDR 30 | Should -Be "255.255.255.252"
    }

    It "Should return 0.0.0.0 for CIDR 0" {
        Convert-CIDRToSubnetMask -CIDR 0 | Should -Be "0.0.0.0"
    }

    It "Should return 255.255.255.255 for CIDR 32" {
        Convert-CIDRToSubnetMask -CIDR 32 | Should -Be "255.255.255.255"
    }
}
```
But aren't the values and their masks known to all, it's just a list of 32 specific options after all.

Before we tackle that idea, let's take a step back from testing, and ask; **Q:** Why write a function to calculate subnet mask values at all, when you could just return the correct value by directly matching the input?

I've not found a good reason not to, so let's see how each option plays out. Perhaps facts will decide?

## Hard-Coding with Switch: A Better Approach?

Should you hard-code the subnet masks?

This hard-code approach defines every subnet mask option by matching the users input via a [Switch](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_switch) statement:
```powershell
function Convert-CIDRToSubnetMask_Switch {
    param (
        [Parameter(Mandatory,ValueFromPipeline)]
        [ValidateRange(0, 32)]
        [int]$CIDR
    )
    # Return the value via a switch block
    switch ($CIDR) {
        0  { "0.0.0.0" }
        1  { "128.0.0.0" }
        2  { "192.0.0.0" }
        3  { "224.0.0.0" }
        4  { "240.0.0.0" }
        5  { "248.0.0.0" }
        6  { "252.0.0.0" }
        7  { "254.0.0.0" }
        8  { "255.0.0.0" }
        9  { "255.128.0.0" }
        10 { "255.192.0.0" }
        11 { "255.224.0.0" }
        12 { "255.240.0.0" }
        13 { "255.248.0.0" }
        14 { "255.252.0.0" }
        15 { "255.254.0.0" }
        16 { "255.255.0.0" }
        17 { "255.255.128.0" }
        18 { "255.255.192.0" }
        19 { "255.255.224.0" }
        20 { "255.255.240.0" }
        21 { "255.255.248.0" }
        22 { "255.255.252.0" }
        23 { "255.255.254.0" }
        24 { "255.255.255.0" }
        25 { "255.255.255.128" }
        26 { "255.255.255.192" }
        27 { "255.255.255.224" }
        28 { "255.255.255.240" }
        29 { "255.255.255.248" }
        30 { "255.255.255.252" }
        31 { "255.255.255.254" }
        32 { "255.255.255.255" }
    }
}
```
## Hard-Coding with Hashtable
Another hard-coded approach defines every subnet mask option by matching the users input via [Hashtable](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_hash_tables):
```powershell
function Convert-CIDRToSubnetMask_Hashtable {
    param (
        [Parameter(Mandatory,ValueFromPipeline)]
        [ValidateRange(0, 32)]
        [int]$CIDR
    )
    $masks = @{
        0  = "0.0.0.0"
        1  = "128.0.0.0"
        2  = "192.0.0.0"
        3  = "224.0.0.0"
        4  = "240.0.0.0"
        5  = "248.0.0.0"
        6  = "252.0.0.0"
        7  = "254.0.0.0"
        8  = "255.0.0.0"
        9  = "255.128.0.0"
        10 = "255.192.0.0"
        11 = "255.224.0.0"
        12 = "255.240.0.0"
        13 = "255.248.0.0"
        14 = "255.252.0.0"
        15 = "255.254.0.0"
        16 = "255.255.0.0"
        17 = "255.255.128.0"
        18 = "255.255.192.0"
        19 = "255.255.224.0"
        20 = "255.255.240.0"
        21 = "255.255.248.0"
        22 = "255.255.252.0"
        23 = "255.255.254.0"
        24 = "255.255.255.0"
        25 = "255.255.255.128"
        26 = "255.255.255.192"
        27 = "255.255.255.224"
        28 = "255.255.255.240"
        29 = "255.255.255.248"
        30 = "255.255.255.252"
        31 = "255.255.255.254"
        32 = "255.255.255.255"
    }
    # return the value via a hashtable lookup
    $masks[$CIDR]
}
```

## Not Hard-Coding - Bitwise calculation
This is the final version improved after consulting the LLM.
```powershell
function Convert-CIDRToSubnetMask_Bitwise {
    param (
        [Parameter(Mandatory,ValueFromPipeline)]
        [ValidateRange(0,32)]
        [int]$CIDR
    )
    $bin = ('1' * $CIDR).PadRight(32, '0')
    $octets = $bin -split '(.{8})' -match '\d'
    ($octets | ForEach-Object { [convert]::ToInt32($_, 2) }) -join '.'
}
```
```powershell
# note: The final line was originally two lines; I avoid the variable assignment for better speed.
# $subnetMask = ($octets | ForEach-Object { [convert]::ToInt32($_, 2) }) -join '.'
# $subnetMask
```

At this point were not yet ready for the testing, before we do that, let's assess the functions on structure.

## Pros of Hard-Coding with Switch
✅ Readability – The mapping is explicit, making it easy to understand. <br>
❌ More Code – Hardcoding makes the script longer. <br>
✅ Cosmetics – although long, it looks good, with good structure.
## Pros of Hard-Coding with Hashtable
✅ Readability – The mapping remains explicit, and is easy to understand. <br>
❌ More Code – Hardcoding makes the script longer. <br>
✅ Cosmetics – similarly long, looks good, with good structure.
## Pros of Not Hard-Coding - Calculation
❌ Readability – The code is less easy for a beginner to understand when not explicitly mapped. <br>
✅ Less Code – who doesn't love a compact, sharp and neat function? <br>
✅ Cosmetics – compact and looks good, and with good structure.

## Performance Comparison: Hard-Coding vs. Calculation

Let's return to the original question now.

**Q:** Why write a function to calculate subnet mask values, when you could just return the correct value by directly matching the input?

By testing the performance of each method, the completion times for each test should help answer that question.

### Benchmarking Execution Times

The simplest test of performance uses [Measure-Command](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/measure-command)

```powershell
Measure-Command { Convert-CIDRToSubnetMask_Switch -CIDR 24 }
Measure-Command { Convert-CIDRToSubnetMask_Hashtable -CIDR 24 }
Measure-Command { Convert-CIDRToSubnetMask_Bitwise -CIDR 24 }
```
### Benchmarking Results
- Switch lookup: ~1.2604ms
- Hashtable lookup: ~0.9555ms
- Bitwise method: ~1.5305ms
- just 0.575ms from slowest to fastest

### Which Approach is Faster?
The lookup method (switch or hash table) is free from complexity. It's essentially a dictionary lookup, making it faster for frequent queries.
The calculation method is more complex, but incurs a small CPU overhead due to shifting and bitwise operations.

For individual runs, the difference is negligible.

So lets ask about test coverage of all 32 options. How would you do that?

To answer that question we return to Pester unit testing we had briefly mentioned.

Pester is a testing and mocking framework for PowerShell, and we can use the testing framework to benchmark the function variants.

Pester supports a [data driven test](https://pester.dev/docs/usage/data-driven-tests) mechanism that is used here solely to avoid having to hard-code all 32 options.

```powershell
Describe "ConvertFromCIDR - Data driven tests" {

    $masks = @{
        0  = "0.0.0.0";         1 = "128.0.0.0";        2  = "192.0.0.0";
        3  = "224.0.0.0";       4  = "240.0.0.0";       5  = "248.0.0.0";
        6  = "252.0.0.0";       7  = "254.0.0.0";       8  = "255.0.0.0";
        9  = "255.128.0.0";     10 = "255.192.0.0";     11 = "255.224.0.0";
        12 = "255.240.0.0";     13 = "255.248.0.0";     14 = "255.252.0.0";
        15 = "255.254.0.0";     16 = "255.255.0.0";     17 = "255.255.128.0";
        18 = "255.255.192.0";   19 = "255.255.224.0";   20 = "255.255.240.0";
        21 = "255.255.248.0";   22 = "255.255.252.0";   23 = "255.255.254.0";
        24 = "255.255.255.0";   25 = "255.255.255.128"; 26 = "255.255.255.192";
        27 = "255.255.255.224"; 28 = "255.255.255.240"; 29 = "255.255.255.248";
        30 = "255.255.255.252"; 31 = "255.255.255.254"; 32 = "255.255.255.255"
    }

    It "CIDR \<CIDR> converts to <Expected>" -ForEach (
        $masks.GetEnumerator() | ForEach-Object {
            @{ CIDR = $_.Key; Expected = $_.Value }
        }
    ) { $CIDR | Convert-CIDRToSubnetMask | Should -Be $Expected }
    # replace Convert-CIDRToSubnetMask for correct function name when testing.

}
```
If you prefer to not use the `Data driven tests` method above, this second example provides a test block containing 6 of the 32 options using the more familiar Pester test structure. This can be expanded to include all 32 test options.

```powershell
Describe "ConvertFromCIDR tests" {

    It "The correct subnet mask is generated" {
        0  | Convert-CIDRToSubnetMask | Should -BeExactly '0.0.0.0'
        1  | Convert-CIDRToSubnetMask | Should -BeExactly '128.0.0.0'
        16 | Convert-CIDRToSubnetMask | Should -BeExactly '255.255.0.0'
        23 | Convert-CIDRToSubnetMask | Should -BeExactly '255.255.254.0'
        24 | Convert-CIDRToSubnetMask | Should -BeExactly '255.255.255.0'
        26 | Convert-CIDRToSubnetMask | Should -BeExactly '255.255.255.192'
    }
}
```

### Test Execution Times - for the Data driven tests method
- Switch lookup: Tests completed in 167ms
- Hashtable lookup: Tests completed in 157ms
- Bitwise method: Tests completed in 177ms
- 20ms from slowest to fastest (performance difference is exacerbated when executing 32 tests)

## Final Verdict
- **Performance:** When performance matters, the hardcoded lookup is slightly faster.
- **Maintainability:** The hardcoded lookup wins here as well, it's more readable and friendly.
- **Flexibility:** While flexibility can be important, it’s not a key factor for this exercise.
- **Bitwise Calculation:** The bitwise method is compact, but slightly slower due to string operations and conversions. It's less intuitive, especially for those unfamiliar with binary IP calculations.

I'd chose the *wrong one* of course, as I would choose the compact code ahead of the faster 🚀 hard coded methods, and the bitwise method is straightforward to follow.

Overall, the difference is minimal unless you're working at large scale.  The choice is yours.
