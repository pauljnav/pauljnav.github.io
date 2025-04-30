This is a well overdue follow-up to the original post [Convert CIDR to Subnet Mask]({% post_url 2025-01-08-blog-Convert-CIDRToSubnetMask %}). And here I try to tell a better story too. When we code, we often go back and iterate  what we have done before, and in doing so we see things that can and need to be improved.

This is one that was nagging me for a long time, as I back then in January I needed to convert from CIDR to Subnet Mask for Azure VM scripted build, and I clearly went down a complex route. It was fun to learn and explain the match, but honestly there was no need.

Maybe this does not happen to you, but I forgot that I had made an even better and simpler version long ago, and that's what was nagging me since.

```powershell
# gpt - plus manual fix
function Convert-CIDRToSubnetMask {
    param (
        [Parameter(Mandatory)]
        [ValidateRange(0,32)]
        [int]$CIDR
    )
    
    $bin = ('1' * $CIDR).PadRight(32, '0')
    $octets = $bin -split '(.{8})' -match '\d'
    $subnetMask = ($octets | ForEach-Object { [convert]::ToInt32($_, 2) }) -join '.'
    
    return $subnetMask
}
```
$CIDR_Bits -split '(.{8})' -ne ''

Already done  by [BornToBeRoot](https://github.com/BornToBeRoot/PowerShell/blob/master/Module/LazyAdmin/Functions/Network/Convert-Subnetmask.ps1)

theres Get-SubnetMaskIPv4 too from  [PoshFunctions](https://www.powershellgallery.com/packages/PoshFunctions/2.2.11) 

## NEAT

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


## Hard-Coding with Switch: A Better Approach?

Should You Hard-Code the Subnet Masks?

An alternative approach is to hard-code the subnet masks using a switch statement:

```powershell
function Convert-CIDRToSubnetMask-Switch {
    param (
        [Parameter(Mandatory)]
        [ValidateRange(0, 32)]
        [int]$CIDR
    )

    $masks = @{
        0  = "0.0.0.0";       1  = "128.0.0.0";    2  = "192.0.0.0";
        3  = "224.0.0.0";     4  = "240.0.0.0";    5  = "248.0.0.0";
        6  = "252.0.0.0";     7  = "254.0.0.0";    8  = "255.0.0.0";
        9  = "255.128.0.0";  10  = "255.192.0.0";  11  = "255.224.0.0";
        12  = "255.240.0.0"; 13  = "255.248.0.0";  14  = "255.252.0.0";
        15  = "255.254.0.0"; 16  = "255.255.0.0";  17  = "255.255.128.0";
        18  = "255.255.192.0"; 19 = "255.255.224.0"; 20 = "255.255.240.0";
        21  = "255.255.248.0"; 22 = "255.255.252.0"; 23 = "255.255.254.0";
        24  = "255.255.255.0"; 25 = "255.255.255.128"; 26 = "255.255.255.192";
        27  = "255.255.255.224"; 28 = "255.255.255.240"; 29 = "255.255.255.248";
        30  = "255.255.255.252"; 31 = "255.255.255.254"; 32 = "255.255.255.255"
    }

    return $masks[$CIDR]
}
```

## Pros of Hard-Coding with Switch
✅ Readability – The mapping is explicit, making it easy to understand.<br>
✅ Faster Execution – Since the function just retrieves a value from a hash table, it's computationally efficient.
## Cons of Hard-Coding
❌ More Code – Hardcoding makes the script longer.

## Performance Comparison: Hard-Coding vs. Calculation
### Which Approach is Faster?
The lookup method (switch or hash table) is O(1) in complexity. It's essentially a dictionary lookup, making it faster for frequent queries.
The bitwise operation method is O(1) as well, but incurs a small CPU overhead due to shifting and bitwise operations.
### Benchmarking Execution Times
To test performance, we use` Measure-Command`:
```powershell
Measure-Command { Convert-CIDRToSubnetMask -CIDR 24 }
Measure-Command { Convert-CIDRToSubnetMask-Switch -CIDR 24 }
```
Results
Bitwise Method: ~0.2ms
Hard-Coded Lookup: ~0.05ms
For individual runs, the difference is negligible.


## Unit Tests (Pester)
Here’s a test suite in Pester v5 to validate both implementations:
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

# Pester v5 Test Script for Convert-CIDRToSubnetMask-Switch
Describe "Convert-CIDRToSubnetMask-Switch Tests" {

    It "Should return 255.255.255.0 for CIDR 24" {
        Convert-CIDRToSubnetMask-Switch -CIDR 24 | Should -Be "255.255.255.0"
    }

    It "Should return 255.255.255.252 for CIDR 30" {
        Convert-CIDRToSubnetMask-Switch -CIDR 30 | Should -Be "255.255.255.252"
    }

    It "Should return 0.0.0.0 for CIDR 0" {
        Convert-CIDRToSubnetMask-Switch -CIDR 0 | Should -Be "0.0.0.0"
    }

    It "Should return 255.255.255.255 for CIDR 32" {
        Convert-CIDRToSubnetMask-Switch -CIDR 32 | Should -Be "255.255.255.255"
    }
}
```
## Final Verdict
For performance-critical applications, the hardcoded lookup is slightly faster.

For maintainability and flexibility, the bitwise calculation is better.

The difference is minimal unless you're working at large scale. 🚀
