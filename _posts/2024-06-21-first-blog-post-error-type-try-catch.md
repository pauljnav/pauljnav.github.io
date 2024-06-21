## How and why did I start a blog?

Well, I'm heavily influenced by Doug Finke, Jeff Hicks, Andrew Pla and many others, and those learned experts often mention that a personal blog is
a great way to help you remember your own tips and tricks during your learning journey with anything that you learn. Just write a blog for yourself, so that is my inspiration.

---

### How to identify the right catch specification for your try-catch blocks.

```powershell
try {
    # Generate the error
    Get-Command not_a_command -ErrorAction Stop
} catch {
    $error[0] | Select Exception
    $error[0] | % Exception
    $error[0] | % CategoryInfo
    $error[0] | % CategoryInfo | % Reason
    $error[0].Exception.GetType().FullName # top tip
    $error[0] | % FullyQualifiedErrorId
}
```

The code in the catch block shows various ways to inspect the error. You see more, pipe from `$error[0]` to `Get-Member` like this: `$error[0] | Get-Member`

#### What is `$error[0]` ?

`$error[0]` an automatic variable. For more details, check the [about_automatic_variables documentation](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_automatic_variables)

####top tip
I show you `$error[0].Exception.GetType().FullName`.

The `Get-Command not_a_command` that I use above generates an `ErrorRecord`, because the term 'not_a_command' is not recognized as the name of a cmdlet, function, or operable program.

The `ErrorRecord` exception type is "CommandNotFoundException", but to catch that exception by type it is not enough to use "CommandNotFoundException".

####What should you use to catch that exception by type?
That answer can be found by reading the `System.RuntimeType` FullName for the Exception.
This value is provided by `$error[0].Exception.GetType().FullName` and it's value is `System.Management.Automation.CommandNotFoundException`

### Here is an example of how you can use that type information to Catch that exception.

```powershell
try {
    # Generate the error
    Get-Command not_a_command -ErrorAction Stop
}
catch [System.Management.Automation.CommandNotFoundException] {
    "A CommandNotFoundException was thrown"
}

```

Play around with the code above, see how commenting out `-ErrorAction Stop` denies the explicit catch handling.
