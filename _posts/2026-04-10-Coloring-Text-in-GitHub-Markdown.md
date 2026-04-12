# Colouring PR Text in GitHub Markdown - A Tip to Stumble Upon

## Summary:
While discussing the argument completer for an AI Provider in PR dfinke/psaisuite#49, I wanted some simple colored text in my PR comments to accurately reflect [PowerShell](https://learn.microsoft.com/powershell/) output.

I found an approach that is rarely documented and perhaps not widely known in the [GitHub community](https://github.com/community).

## Why did I want colour?

When documenting shell output or sharing UX patterns, I find that text colour is incredibly useful to visually distinguish prompts, commands, variables, and warnings.

I didn't want the blandness of this:
- Yellow for commands
- Grey for parameters
- Green for variables and comments
- White for markdown headers
- Orange for warnings

I wanted colours!
  - $\textcolor{Yellow}{\textrm{Yellow}}$ for commands 
  - $\textcolor{Grey}{\textsf{Grey}}$ for parameters
  - $\textcolor{Green}{\textsf{Green}}$ for variables and comments
  - $\textcolor{White}{\textsf{White}}$ for markdown headers
  - $\textcolor{Orange}{\textsf{Orange}}$ for warnings

And
<summary>
   I wanted
   $\textcolor{yellow}{\textsf{to mix}}$
   $\textcolor{grey}{\textsf{ colours}}$
   $\textcolor{green}{\textsf{ on a}}$
   $\textcolor{blue}{\textsf{ single}}$
   $\textcolor{white}{\textsf{ line,}}$
   $\textcolor{green}{\textsf{ just like the}}$
   $\textcolor{white}{\textsf{ shell output}}$
</summary>

---

Standard [GitHub-flavored markdown](https://github.github.com/gfm/) doesn't support text colour in PR comments, not in any obvious or "official" way.  

---

### Failed Attempts and Odd Behaviors
Before landing on the `LaTeX` trick, I tried several approaches that *should* have worked in theory but didn't:

- **Inline HTML** - GitHub strips CSS `style=` attributes entirely - not only for PR Comments.

Even `<span style="color:yellow">` becomes plain text.  
The following outputs:  `Yellow for prompts`
```html
<span style="color: Yellow;">Yellow</span> for prompts
```

- The basic `<font>` tags - Rendered as literal text, not HTML.  
- The following outputs as:  `Yellow for prompts`
```html
<font color="Yellow">Yellow</font> for prompts
```

- **LaTeX outside `<summary>`** - GitHub renders it as plain text in normal Markdown, but *inside* a `<summary>` block it
<summary>
   $\textcolor{green}{\textsf{suddenly}}$
   $\textcolor{yellow}{\textsf{ becomes}}$
   $\textcolor{red}{\textsf{ styled}}$
</summary>

---

Internet examples using `html` and other methods provided for multiple colours, but only one colour on an individual line.  
Some methods gave many colours on one line, but they did not colour the Verb-Noun correctly due to the presense of the `hyphen` char.
```
${\color{red}Color \space your \space \color{green}Verb-Noun \space \color{blue}in \space Github}$
```
Outputs as: ${\color{red}Color \space your \space \color{green}Verb-Noun \space \color{blue}in \space Github}$  
Nice colours, but that injects spaces before and after the verb noun hyphen (notOK).

---

These failures helped narrow down the little corner of GitHub's rendering engine where LaTeX color commands are interpreted.

---

## An Accidental Discovery: LaTeX Color Commands

I found some interesting references:
- [stackoverflow how-to-add-color](https://stackoverflow.com/questions/11509830/how-to-add-color-to-githubs-readme-md-file)  
- github markup issue github/markup/issues/369
- pvrego gist [How to make README.md files with colored texts in Github](https://gist.github.com/pvrego/2e346674c3abbaa6366dfe86b8488dc9)

---

This is what I wanted for the PR comment.

By dropping in LaTeX-like color commands inside `<summary>` blocks, like this:

```md
<summary>
   $\textcolor{yellow}{\textsf{Invoke-ChatCompletion}}$
   $\textcolor{grey}{\textsf{-Messages}}$
   $\textcolor{green}{\textsf{\$Message}}$
   $\textcolor{white}{\textsf{fireworksai:}}$
   $\textcolor{yellow}{\textsf{WARNING: FireworksAIKey environment variable is not set..}}$
</summary>
```

You get an ideal mimic for the PowerShell output:

<summary>
   $\textcolor{yellow}{\textsf{Invoke-ChatCompletion}}$
   $\textcolor{grey}{\textsf{-Messages}}$
   $\textcolor{green}{\textsf{\$Message}}$
   $\textcolor{white}{\textsf{fireworksai:}}$
   $\textcolor{yellow}{\textsf{WARNING: FireworksAIKey environment variable is not set..}}$
</summary>

---

[GitHub Reference](https://docs.github.com/en/get-started/writing-on-github/working-with-advanced-formatting/writing-mathematical-expressions)
GitHub's math rendering capability uses MathJax; an open source, JavaScript-based display engine. MathJax supports a wide range of LaTeX macros.  
Mathematical expressions rendering is available in GitHub Issues, GitHub Discussions, pull requests, wikis, and Markdown files.

## The Goodies

I hope the `goodies` documented here can help you in some of your own PR efforts.

---

## Take the Markdown course from [Microsoft](https://learn.microsoft.com) (and more)

[Learn to use Markdown](https://learn.microsoft.com/en-us/training/modules/communicate-using-markdown/)

[Other GitHub Platform courses](https://learn.github.com/courses?product=GitHub+Platform)

[Get started with GitHub documentation](https://docs.github.com/en/get-started)

---

## Limitations & Gotchas

- Works in `<summary>` blocks  
- Works in README.md  
- Works in issues
- Be aware of markdown rendering differs between:
  - GitHub Web
  - VS Code

---

**Summary:**
I stumbled onto this “feature” while looking for a better documentation experience, and even though it isn't fully supported in GitHub markdown, it's a curious workaround when you need to show your intentions! Try it; copy markdown with LaTeX color commands into VS Code (with a LaTeX extension) or into GitHub, and watch it light up!.