# The Mock That Mocked Itself: Stabilising `ImportExcel` Tests in Pester

## TL;DR
A failing Pester test was **not** caused by `TestDrive:\` paths. The real issue was a recursive mock pattern:

- `Get-DemoValue` (an anonymised stand-in name) calls `ImportExcel\Open-ExcelPackage -Path ...`
- The test mocked `Open-ExcelPackage`
- Inside that mock, it called `ImportExcel\Open-ExcelPackage -Create ...`
- Pester intercepted that call too, creating recursion and test failures.
- The package never initialised, then `Close-ExcelPackage` received `$null`

In this article, `Get-DemoValue` is used intentionally to avoid exposing production identifiers.

The stable fix combines two moves:

1. Use a **typed `New-MockObject`** for `OfficeOpenXml.ExcelPackage` so tests avoid real workbook creation.
2. Add a **null guard** in the function `finally` block before closing the package.

---

## The Symptom
The focused test failed with:

- `ScriptCallDepthException: The script failed due to call depth overflow`
- `ParameterBindingValidationException: Cannot bind argument to parameter 'ExcelPackage' because it is null.`

In practice, the call-depth exception showed up first, and the null `ExcelPackage` binding error appeared as a downstream symptom after package creation failed. At first glance this looked like a path/provider issue (`TestDrive:\test.xlsx`). But targeted probes showed `ImportExcel\Open-ExcelPackage -Create -Path 'TestDrive:\test.xlsx'` works fine.

---

## Root Cause: Recursive Mock Interception
In Pester, a mock can still intercept calls that appear module-qualified. That matters because this pattern is dangerous:

```powershell
Mock -CommandName Open-ExcelPackage -MockWith {
    ImportExcel\Open-ExcelPackage -Create -Path $testPath
}
```

When recursion occurs, execution can overflow with `ScriptCallDepthException`. If the flow then reaches cleanup, no usable package has been produced, and `finally` executes close logic with `$null`, surfacing the `ExcelPackage` binding error as a secondary symptom.

---

## Minimal Reproduction You Can Paste
If you want a tiny, dependency-free repro for training or onboarding, this snippet captures the same danger pattern. The assertion intentionally uses `Should -Not -Throw` so the failure is obvious in a demo.

```powershell
#region functions

function Invoke-Leaf {
    'real-value'
}

function Get-DemoValue {
    [CmdletBinding()]
    param()
    Invoke-Leaf
}
#endregion

#region tests
Describe 'Recursive mock danger repro' {
    It 'can overflow when a mock calls the same mocked command' {
        Mock -CommandName Invoke-Leaf -MockWith {
            Invoke-Leaf   # <- recursively re-enters the same mock
        }

        { Get-DemoValue } | Should -Not -Throw # It will throw
    }
}
#endregion
```

What this demonstrates:

- Mock interception can re-enter the same mocked command
- Recursive mock bodies can produce `ScriptCallDepthException`
- The failure often appears far from the original test intent

---

## The Stabilised Strategy
### 1) Prefer `New-MockObject` for a typed package
Create a strongly typed `OfficeOpenXml.ExcelPackage` test double and inject only the member graph the function needs (`Workbook.Worksheets.Name`).

```powershell
# Mock strategy notes:
# - Keep this as a unit test by avoiding real workbook/file creation.
# - Return a strongly typed ExcelPackage test double for parameter binding.
# - Shape only the member path Get-DemoValue reads: Workbook.Worksheets.Name.
$mockWorkbook = [pscustomobject]@{
    Worksheets = [pscustomobject]@{ Name = @('Sheet1', 'Sheet2') }
}
$mockExcelPackage = New-MockObject -Type 'OfficeOpenXml.ExcelPackage' -Properties @{
    Workbook = $mockWorkbook
}

Mock -CommandName Open-ExcelPackage -MockWith { $mockExcelPackage }
```

Why this is robust:

- Returns a strongly typed `OfficeOpenXml.ExcelPackage`
- Satisfies `Import-Excel -ExcelPackage` strong typing
- Avoids recursive self-mocking entirely
- Avoids dependency on creating a workbook/file
- Keeps tests fast and focused on function behavior

### 1b) Alternative: parameter-filtered pass-through mock
If you prefer closer integration behavior, you can still use the parameter-filtered strategy to let `-Create` call the real command.

```powershell
# Mock strategy notes:
# - Get-DemoValue calls ImportExcel\Open-ExcelPackage without -Create.
# - If this mock calls Open-ExcelPackage again without filtering, it recurses into itself.
# - We therefore mock only non--Create calls and, inside the mock body,
#   call the real module command with -Create to build a real ExcelPackage.
# - Returning a real package keeps Import-Excel parameter binding happy.
Mock -CommandName Open-ExcelPackage -ParameterFilter { -not $Create } -MockWith {
    $excelPackage = ImportExcel\Open-ExcelPackage -Create -Path $testPath
    $testWorksheets | ForEach-Object {
        ImportExcel\Add-Worksheet -ExcelPackage $excelPackage -WorksheetName $_ | Out-Null
    }
    return $excelPackage
}
```

Why this is useful to learn (and still valid):

- Demonstrates `-ParameterFilter` as a clean recursion guard
- Preserves realistic package behavior for integration-leaning tests
- Useful when test intent includes verifying workbook construction flow

Tradeoffs compared to `New-MockObject`:

- Slightly higher dependency on `ImportExcel` runtime behavior and file creation path
- More moving parts than a narrow typed double

### 2) Keep `Import-Excel` mocked at payload level

```powershell
Mock -CommandName Import-Excel -MockWith {
    param($excelPackage, $WorksheetName)
    return "Data from $WorksheetName"
}
```

This isolates workbook content while preserving flow through `Get-DemoValue`.

### 3) Add a null-safe `finally` in production code

```powershell
finally {
    if ($null -ne $package) {
        ImportExcel\Close-ExcelPackage -ExcelPackage $package -NoSave
    }
}
```

This prevents cleanup from masking upstream failures and improves diagnosability.

---

## Why `New-MockObject` Works Well Here
`New-MockObject` is a strong fit for this case because `Get-DemoValue` only needs a narrow slice of package behavior:

- typed binding for `-ExcelPackage`
- `Workbook.Worksheets.Name` lookup for sheet validation

By shaping just those members, the tests avoid external file creation while preserving the contract that matters for this function.

---

## Learning Opportunity: Keep Both Patterns in Your Toolkit
For this codebase, `New-MockObject` is the best default for fast, dependency-light unit tests.

The parameter-filtered mock remains an excellent teaching and troubleshooting pattern when:

- you need realistic object behavior from the external module,
- you want to validate call-shape branching (`-Create` vs non-`-Create`), or
- you are debugging mock interception edge cases in Pester.

---

## Result
After this strategy:

- Focused failure path passes
- Full test file passes (`12` passing tests)
- Mocks document intent clearly
- Cleanup logic is resilient

---

## Practical Pattern You Can Reuse
When mocking strongly typed dependencies:

1. Mock the boundary command (`Open-ExcelPackage`) to return a typed double
2. Shape only the properties your unit touches
3. Keep downstream payload commands (`Import-Excel`) mocked with deterministic returns
4. Add defensive cleanup in production `finally` blocks

That pattern reduces external dependency surface, preserves type fidelity, and keeps tests deterministic.
