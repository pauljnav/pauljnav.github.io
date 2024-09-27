# Decoding an encoded URI with .Net methods

The other day while working with an SSRS server, I found that the 'Data feed' option was returning an encoded uri string that did not work when used as a uri in my browser.
The parts causing issue were the following:
`&amp;rs%3AFormat=ATOM`
`&amp;rc%3AItemPath=table`

Varied methods exist, so here are the comparisons for each example

```Powershell
$string = '&amp;rs%3AFormat=ATOM&amp;rc%3AItemPath=table'
[System.Net.WebUtility]::UrlDecode($string)   # &amp;rs:Format=ATOM&amp;rc:ItemPath=table
[System.Web.HttpUtility]::UrlDecode($string)  # &amp;rs:Format=ATOM&amp;rc:ItemPath=table
[System.Uri]::UnescapeDataString($string)     # &amp;rs:Format=ATOM&amp;rc:ItemPath=table
[System.Web.HttpUtility]::HtmlDecode($string) # &rs%3AFormat=ATOM&rc%3AItemPath=table
[System.Net.WebUtility]::HtmlDecode($string)  # &rs%3AFormat=ATOM&rc%3AItemPath=table
```

### Target behaviour
#### Decoding the ':' character
Decode `&amp;` to `&`
#### Decoding the ':' character
Decode `rs%3A` to `rs:`

### Review
The tests show that none of the methods decode all parts.

The first two using `UrlDecode()` methods behaved the same, both did not decode `&amp;` and they have decoded `:` correctly.

The third using `UnescapeDataString()` method also behaved the same, not not decoding `&amp;` and decoding `:` correctly.

The last two using `HtmlDecode()` methods switched the result around, as it decoded `&amp;` to `&`, and did not decode `%3A` to `:`

### To conclude
You need to combine `UrlDecode()` and `HtmlDecode()` methods, and here are some suggestions.

#### Combine using a pipeline
```Powershell
$string = '&amp;rs%3AFormat=ATOM&amp;rc%3AItemPath=table'
$string | ForEach-Object {[System.Net.WebUtility]::HtmlDecode($_)} | ForEach-Object {[System.Net.WebUtility]::UrlDecode($_)}
# using the ForEach-Object alias '%'
$string | % {[System.Net.WebUtility]::HtmlDecode($_)} | % {[System.Net.WebUtility]::UrlDecode($_)}
```
#### Combine by nesting one method inside the other method
```Powershell
$string = '&amp;rs%3AFormat=ATOM&amp;rc%3AItemPath=table'
[System.Net.WebUtility]::UrlDecode([System.Net.WebUtility]::HtmlDecode($string))
# Switching the order has the same result
[System.Net.WebUtility]::HtmlDecode([System.Net.WebUtility]::UrlDecode($string))
```

### Why System.Net.WebUtility and not System.Web.HttpUtility ?

System.Net.WebUtility is recommended over System.Web.HttpUtility because System.Net.WebUtility is part of the runtime, while System.Web.HttpUtility requires the System.Web.HttpUtility.dll assembly.

#### Microsoft suggest:

> To encode or decode values outside of a web application, use the WebUtility class.

> The System.Uri class also contains methods and properties that can be used for similar purposes.

The PowerShell ISE has System.Web.HttpUtility as part of it's runtime, but it's more correct to align with Windows PowerShell or Windows PowerShell Core.

### References
https://learn.microsoft.com/en-us/dotnet/api/system.net.webutility

https://learn.microsoft.com/en-us/dotnet/api/system.web.httputility

https://learn.microsoft.com/en-us/dotnet/api/system.uri
