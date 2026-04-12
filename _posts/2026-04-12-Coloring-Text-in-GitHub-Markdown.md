# Colouring PR Text in GitHub Markdown - A Tip to Stumble Upon

While discussing the argument completer for an AI Provider in PR [dfinke/psaisuite#49](https://github.com/dfinke/psaisuite/pull/49), I wanted some simple colored text in my PR comments to accurately reflect [PowerShell](https://learn.microsoft.com/powershell/) output.

And then I discovered a trick that is rarely documented and perhaps not widely known in the [GitHub community](https://github.com/community).

## Why did I want colour?

When documenting shell output or sharing UX patterns, colour is incredibly useful to visually distinguish prompts, commands, variables, and warnings.

I didn't want the blandness of this:
- Yellow for commands
- Grey for parameters
- Green for variables and comments
- White for markdown headers
- Orange for warnings

I wanted colours, not a list of colour names!
<ul>
  <li><span style="color: Yellow; background-color: #1e1d1d;">Yellow</span> for commands</li>
  <li><span style="color: Grey; background-color: #1e1d1d;">Grey</span> for parameters</li>
  <li><span style="color: Green; background-color: #1e1d1d;">Green</span> for variables and comments</li>
  <li><span style="color: White; background-color: #1e1d1d;">White</span> for markdown headers</li>
  <li><span style="color: Orange; background-color: #1e1d1d;">Orange</span> for warnings</li>
</ul>

<div style="background-color: #1e1d1d; padding: 6px;">
And I wanted
  <span style="color: yellow;">to mix</span>
  <span style="color: grey;"  >colours</span>
  <span style="color: green;" >on a</span>
  <span style="color: blue;"  >single</span>
  <span style="color: white;" >line,</span>
  <span style="color: green;" >just like the</span>
  <span style="color: white;" >shell output</span>
</div>

---

Standard [GitHub-flavored markdown](https://github.github.com/gfm/) doesn't support text colour in PR comments, not in any obvious or "official" way.  

---

### The trick I discovered was to code the text using `LaTeX` format

I'm not trying to fool you, dear reader, but I must confess that colourful text blocks above are rendered using HTML, not `LaTeX`. GitHub Pages Markdown doesn't support LaTeX colour rendering, so HTML had to step in for this blog post.

*Sorry about that*

But you can sneak the examples below into your `README.md` and PR Comments to correctly display these LaTeX blocks.

## Working LaTeX Examples

```LaTeX
- $\textcolor{Yellow}{\textrm{Yellow}}$ for commands
- $\textcolor{Grey}{\textsf{Grey}}$ for parameters
- $\textcolor{Green}{\textsf{Green}}$ for variables and comments
- $\textcolor{White}{\textsf{White}}$ for markdown headers
- $\textcolor{Orange}{\textsf{Orange}}$ for warnings
```

And for multi‑colour inline text:

```LaTeX
<summary>
   $\textcolor{yellow}{\textsf{to mix}}$
   $\textcolor{grey}{\textsf{ colours}}$
   $\textcolor{green}{\textsf{ on a}}$
   $\textcolor{blue}{\textsf{ single}}$
   $\textcolor{white}{\textsf{ line,}}$
   $\textcolor{green}{\textsf{ just like the}}$
   $\textcolor{white}{\textsf{ shell output}}$
</summary>
```

---

## Why the `<summary>` Tag Matters

In PR comments, GitHub renders LaTeX inline *only inside* `<summary>` tags.  
Without them, each coloured word appears on its own line.

In `README.md`, the `<summary>` tag is optional — the line renders correctly either way.

Try both in a PR comment and in a README to see the difference.

---

## Failed Attempts and Odd Behaviors
Before landing on the `LaTeX` trick, I tried several approaches that *should* have worked in theory but didn't:

### **Inline HTML**

GitHub strips `style=` attributes entirely - and not only for PR Comments.

Even `span style="color: yellow;">Yellow</span>` renders as plain text.

And `<font>` tags are treated just the same as plain.

### **LaTeX in GitHub Pages Markdown**

GitHub Pages does not render LaTeX colour commands.  

This block:
```LaTeX
<summary>
   $\textcolor{green}{\textsf{suddenly}}$
   $\textcolor{yellow}{\textsf{ becomes}}$
   $\textcolor{red}{\textsf{ styled}}$
</summary>
```
displays exactly as the following on this very blog post:
<summary>
   $\textcolor{green}{\textsf{suddenly}}$
   $\textcolor{yellow}{\textsf{ becomes}}$
   $\textcolor{red}{\textsf{ styled}}$
</summary>

---

Internet examples using `html` and other methods provided for multiple colours, but only one colour on an individual line.

Some methods gave many colours on one line, but they did not colour the Verb-Noun correctly due to the presense of the `hyphen` char.

The following example outputs as: ${\color{red}Color \space your \space \color{green}Verb-Noun \space \color{blue}in \space Github}$  
```
   ${\color{red}Color \space your \space \color{green}Verb-Noun \space \color{blue}in \space Github}$
```

This is math‑mode LaTeX using `\color{red}` (a math‑mode LaTeX colour switch). Nice colours, but that injects spaces before and after the verb noun hyphen (notOK). And you get that result when added to PR, README.md and here in GitHub Pages Markdown.

## Text‑Mode LaTeX (the one that works)

The format I blog about is different, this is text‑mode LaTeX using `\textcolor{red}` (a text‑mode colour wrapper).
```
   $\textcolor{red}{\textsf{ styled}}$
```

---

These failures helped narrow down the little corner of GitHub's rendering engine where LaTeX color commands are interpreted.

## An Accidental Discovery: LaTeX Color Commands

I found some interesting references:
- pvrego gist [How to make README.md files with colored texts in Github](https://gist.github.com/pvrego/2e346674c3abbaa6366dfe86b8488dc9)
- [stackoverflow questions/11509830](https://stackoverflow.com/questions/11509830/how-to-add-color-to-githubs-readme-md-file)  
- [github/markup/issues/369](https://github.com/github/markup/issues/369)
- [github/markup/issues/1440](https://github.com/github/markup/issues/1440) remains open, related to the stackoverflow questions/11509830 above.

---

## Finally, this is what I wanted for the PR comment.

By dropping in LaTeX text‑mode colour commands inside `<summary>` blocks, like this:

```LaTeX
<summary>
   $\textcolor{yellow}{\textsf{Invoke-ChatCompletion}}$
   $\textcolor{grey}{\textsf{-Messages}}$
   $\textcolor{green}{\textsf{\$Message}}$
   $\textcolor{white}{\textsf{fireworksai:}}$
   $\textcolor{yellow}{\textsf{WARNING: FireworksAIKey environment variable is not set..}}$
</summary>
```

You get an ideal mimic for the PowerShell output:

<summary style="background-color: #1e1d1d; padding: 6px;">
  <span style="color: yellow;">Invoke-ChatCompletion</span>
  <span style="color: grey;">-Messages</span>
  <span style="color: green;">$Message</span>
  <span style="color: white;">fireworksai:</span>
  <span style="color: yellow;">WARNING: FireworksAIKey environment variable is not set..</span>
</summary>

As a side-note, we didnt implement the <span style="color: yellow;">WARNING on environment var</span> text after discusing on the PR.

---

## A Key GitHub Reference
GitHub's math rendering capability uses MathJax; an open source, JavaScript-based display engine. MathJax supports a wide range of LaTeX macros.  
Mathematical expressions rendering is available in GitHub Issues, GitHub Discussions, pull requests, wikis, and Markdown files.  
See [writing-on-github](https://docs.github.com/en/get-started/writing-on-github/working-with-advanced-formatting/writing-mathematical-expressions)

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

**Final Thoughts:**

I stumbled onto this “feature” while looking for a better documentation experience, and even though it's not fully supported, it works well enough to be useful

Try it. Copy the LaTeX colour commands into VS Code (with a LaTeX extension) or into GitHub, and watch them light up!.