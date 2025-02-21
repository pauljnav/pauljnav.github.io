> :memo: **Note:** After reading the Tip, heres a measure

# Avoid Array Addition - Tip
I'm a fan of [PowerShell.tiPS](https://www.powershellgallery.com/packages/tiPS) by [Daniel Schroeder](https://blog.danskingdom.com/) (aka [deadlydog](https://github.com/deadlydog)), and the other day a really nice tip submitted by: [Santiago Squarzon](https://github.com/santisq) appeared in my [PowerShell](https://learn.microsoft.com/en-us/powershell/scripting/overview) console.
The tip was so nice that I expanded the example and used [Measure-Command](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/measure-command) to test the benefit described by Santiago's tip.

## Firstly, the tip

<p style="font-family:consolas; color:cyan;"></p>

```powershell
# --------------------------- Avoid Array addition [Performance] ---------------------------
# Array addition is an expensive and inefficient operation and can usually be replaced by PowerShell explicit loop assignment.

# Use a `List<T>` instead in those cases when adding to a collection while looping is required.
Example:
# Array addition:
$items = @()
foreach ($i in 0..10) {
    $items += $i
}

# Can be easily replaced with explicit assignment:
$items = foreach ($i in 0..10) {
    $i
}

# And, when not possible, a List<T> is recommended:
$items = [System.Collections.Generic.List[int]]::new()
foreach ($i in 0..10) {
    $items.Add($i)
}
```
More information:
https://learn.microsoft.com/en-us/powershell/scripting/dev-cross-plat/performance/script-authoring-considerations?view=powershell-7.5&viewFallbackFrom=powershell-7.3#array-addition

Tip submitted by: Santiago Squarzon ([santisq](https://github.com/santisq))


## Expanding the example with Measure-Command
PowerShell is full of many great Commands Cmdlets and Modules, and one of my favourites is [Measure-Command](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/measure-command)
`Measure-Command` makes it very easy to gauge the performance of your commands, and to test improvements. SO thats what dont here as the example, I've wrapped the tip with `Measure-Command` to reveal how much faster the code gets when following that great tip.

### The goodies

```powershell
$upperBound = 10000

Measure-Command {
    # Array addition:
    $items = @()
    foreach ($i in 0..$upperBound) {
        $items += $i
    }
} | Select-Object TotalMilliseconds                # TotalMilliseconds: 2076.1273

Measure-Command {
    # Can be easily replaced with explicit assignment:
    $items = foreach ($i in 0..$upperBound) {
        $i
    }
} | Select-Object TotalMilliseconds                # TotalMilliseconds: 15.3416

Measure-Command {
    # And, when not possible, a List<T> is recommended:
    $items = [System.Collections.Generic.List[int]]::new()
    foreach ($i in 0..$upperBound) {
        $items.Add($i)
    }
} | Select-Object TotalMilliseconds                # TotalMilliseconds: 11.2543

TotalMilliseconds
-----------------
        2076.1273 # Array addition
          15.3416 # Explicit assignment
          11.2543 # List<T>

```

_**Tip** there is a great variation of performance measurement at the **[What's New in PowerShell 7.5](https://learn.microsoft.com/en-us/powershell/scripting/whats-new/what-s-new-in-powershell-75)** document mentioned below._


## Performance improvements
See [What's New in PowerShell 7.5](https://learn.microsoft.com/en-us/powershell/scripting/whats-new/what-s-new-in-powershell-75) for some relevant additional info on this topic as PowerShell 7.5-rc.1 included PR#[23901](https://github.com/PowerShell/PowerShell/pull/23901) from [jborean93](https://github.com/jborean93) that improves the performance of the += operation for an array of objects.

