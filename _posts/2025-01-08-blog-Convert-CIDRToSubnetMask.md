# Convert CIDR to subnet mask in PowerShell

### Happy New Year to all
It's 2025, a year we used to associate with science fiction is now a reality, and "still no flying cars"!

It's the beginning of a new year, and I'm back at my usual daily stuff; building labs, virtual machines, creating, testing and solving Jira, etc. And today, I wanted to add a network adapter to a new VM and I asked myself "how would I define MAC address based off having the machines CIDR"?

Well it got me thinking (and also chatting with my LLM) and in this post I'm sharing the end result. A function named `Convert-CIDRToSubnetMask`

I hope you find the goodies useful.
```powershell
function Convert-CIDRToSubnetMask
{
<#
.SYNOPSIS
    Converts a CIDR (Classless Inter-Domain Routing) notation to a subnet mask in dotted decimal format.
.DESCRIPTION
    This function takes an integer representing the CIDR prefix length (0-32) and returns the corresponding subnet mask in dotted decimal format (e.g., 255.255.255.0 for /24).
.PARAMETER CIDR
    The CIDR prefix length, an integer value between 0 and 32.
.OUTPUTS
    String
    The subnet mask in dotted decimal notation (e.g., 255.255.255.0).
.EXAMPLE
    Convert /24 to a subnet mask
    Convert-CIDRToSubnetMask -CIDR 24 # 255.255.255.0
.EXAMPLE
    Convert /16 to a subnet mask
    Convert-CIDRToSubnetMask -CIDR 16 # 255.255.0.0
.EXAMPLE
    Convert /22 to a subnet mask
    Convert-CIDRToSubnetMask -CIDR 22 # 255.255.252.0
.NOTES
    Convert cidr to subnet mask in powershell
    Author: Paul Naughton
    Date: Jan 2025
.LINK
    start https://en.wikipedia.org/wiki/Classless_Inter-Domain_Routing
#>

    param (
        [Parameter(Mandatory)]
        [ValidateRange(0, 32)]
        [Int]$CIDR
    )

    # Generate the subnet mask as an unsigned integer
    $subnetMask = [uint32]([math]::Pow(2, $CIDR) - 1) -shl (32 - $CIDR)
    <#
        [math]::Pow(2, $CIDR)
            raises 2 to the power of $CIDR, forming the base for the subnet mask calculation.
        -1
            Subtracting 1 converts the power of 2^CIDR into a bitmask with $CIDR number of 1s in binary.
            If $CIDR = 24
            2 ^ 24      = 16777216 = 00000001 00000000 00000000 00000000
            16777216 -1 = 16777215 = 00000000 11111111 11111111 11111111
        [uint32]()
            Ensures the value is treated as a 32-bit unsigned integer.
        -shl (32 - $CIDR)
            Left-shifts the 1s in the bitmask to the most significant bits, forming a valid subnet mask.
            32 - 24 = 8
            16777215 after shift left 8: 11111111 11111111 11111111 00000000
    #>

    # Convert the subnet mask to a byte array
    $subnetMaskBytes = [BitConverter]::GetBytes($subnetMask)

    # Handle Endianness explicitly
    if ([BitConverter]::IsLittleEndian) {
        $subnetMaskBytes = $subnetMaskBytes[3..0]
    }

    # Convert the byte array to dotted decimal notation
    $dottedDecimal = ($subnetMaskBytes -join '.')

    # Output the dotted decimal subnet mask
    Write-Output $dottedDecimal
}
```