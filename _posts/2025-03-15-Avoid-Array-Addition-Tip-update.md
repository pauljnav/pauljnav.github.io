## Avoid Array Addition - Tip updated

Like many, I'm a fan of [PowerShell.tiPS](https://www.powershellgallery.com/packages/tiPS) by [Daniel Schroeder](https://blog.danskingdom.com/) (aka [@deadlydog](https://github.com/deadlydog)), and the other day a really nice tip submitted by: Santiago Squarzon [@santisq](https://github.com/santisq) appeared in my [PowerShell](https://learn.microsoft.com/en-us/powershell/scripting/overview) console.
The tip was so sweet that I just had to go ahead and expand the example, and by using [Measure-Command](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/measure-command) was able to reveal the performance benefit described by Santiago's tip.

### PowerShell 7.5 Performance Improvement
The tip covers the performance of array addition, so it's important to mention an improvement for this [operation](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_operators) added to Powershell.

Read [What's New in PowerShell 7.5 #Engine Improvements](https://learn.microsoft.com/en-us/powershell/scripting/whats-new/what-s-new-in-powershell-75#engine-improvements) for the details.

`PowerShell 7.5-rc.1` has included [PR#23901](https://github.com/PowerShell/PowerShell/pull/23901) from [@jborean93](https://github.com/jborean93) that improves the performance of the `+=` operation for an array of objects. The improvement is so good, it negates the need for us powershellers to adjust our code when using the `+=` assignment operator.

_**There's more - a pre-tip tip**: there is a great example of code for measuring [performance improvements](https://learn.microsoft.com/en-us/powershell/scripting/whats-new/what-s-new-in-powershell-75#performance-improvements) at the same [What's New in PowerShell 7.5](https://learn.microsoft.com/en-us/powershell/scripting/whats-new/what-s-new-in-powershell-75) link._

### The tip itself
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

More information:
https://learn.microsoft.com/en-us/powershell/scripting/dev-cross-plat/performance/script-authoring-considerations#array-addition
**Tip submitted by:** Santiago Squarzon (@santisq)
```

### Expanding the example with Measure-Command
PowerShell is full of many great Commands, Cmdlets and Modules and a firm favourite is `Measure-Command`.<br>
[Measure-Command](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/measure-command) makes it easy to assess the performance of your commands/scripts/codeblocks and to test improvements. So that's what I show in the example. By wrapping the code blocks with `Measure-Command` I reveal how much faster the code gets for each of it's examples.

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
```
### The scores / results
```powershell
 Powershell 7.4.5    Windows Powershell 5.1   Windows PowerShell ISE 5.1
TotalMilliseconds         TotalMilliseconds            TotalMilliseconds
-----------------         -----------------            -----------------
          798.65                 1235.1058                    1313.8209     # Array addition
            3.77                   10.7214                       6.9199     # Explicit assignment
         11.9616                    7.5251                        9.739     # List<T>
```
I'm not yet using PowerShell 7.5, so I turned to GitHub Actions for help to measure the performance of the 3 blocks on Powershell 7.5.0 and I'm impressed.
```powershell
Powershell 7.5 (7.5.0-1.deb) on "Ubuntu 24.04.2 LTS"
TotalMilliseconds
           62.98
            6.85
           17.60
```

### Measured Improvements
Powershell 5.1 sees a reduction from 1235.10 milliseconds for `Array addition` to 7.52 for `List<T>`.

Powershell 7.4 has a reduction from 798.65 milliseconds to 11.9616 which is every bit as good.  Anyone surprised to see the `List<T>` go up from `v5.1` 7.52 to `v7.4` 11.96 ?

Then along strolls [@deadlydog](https://github.com/deadlydog) and gives us a far faster `Array addition` with `v7.5`. Chapeau!

My typical usage matches the explicit assignment method. It's the closest to assigning the multiple values explicitly when defining an object having multiple values from the outset and that is naturally quick.

Examples
`[string[]]$values = 'A','B','C','D','E'` or `[string[]]$values = @('A','B','C','D','E')`.

The slowest method is slow because it starts by assigning only the first value, then adding each value in sequence;
```powershell
[string[]]$values='A'
$values = $values+'B'
$values = $values+'C'
$values = $values+'D'
$values = $values+'E'`
```
**That code even looks slow!**

My thanks to the members of PowerShell community for feedback on [original post]([_posts\2025-02-21-Avoid-Array-Addition-Tip.md), especially [@JustinGrote](https://bsky.app/profile/posh.guru)

P.S. I had to learn a little more GitHub Actions for this blog update as I needed the runner to perform a PowerShell upgrade to 7.5.0 which required the action to setup-homebrew `uses: Homebrew/actions/setup-homebrew@master`. (Perhaps a post on that someday 😀😀😀 )
