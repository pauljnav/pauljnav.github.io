# Colouring PR Text in GitHub Markdown - A Tip to Stumble Upon

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

I'm not trying to fool you, dear reader, but I must confess that colourful text blocks above are actually rendered using HTML, not `LaTeX`. This blog post written in GitHub Pages Markdown doesn't support LaTeX formatting, so HTML had to be used in the blog to display the idea.

*Sorry about that!*

Use the examples below to sneak and correctly display both LaTeX blocks into your `README.md` and PR Comments.

```LaTeX
I wanted colours!
- $\textcolor{Yellow}{\textrm{Yellow}}$ for commands
- $\textcolor{Grey}{\textsf{Grey}}$ for parameters
- $\textcolor{Green}{\textsf{Green}}$ for variables and comments
- $\textcolor{White}{\textsf{White}}$ for markdown headers
- $\textcolor{Orange}{\textsf{Orange}}$ for warnings
```

```LaTeX
<summary>
   And I wanted
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

Now I hear you asking; "What is the `<summary>..</summary>` tags for?"

The `<summary>` tags are needed in a PR comment to make the sentence appear on one line, and without it your texts get written on many lines.  
But for a README.md, the `<summary>` tags can be omitted as the sentence appears on one line in either case.

Try both out in GitHub within a PR comment and README.md.

---

### Failed Attempts and Odd Behaviors
Before landing on the `LaTeX` trick, I tried several approaches that *should* have worked in theory but didn't:

#### **Inline HTML**

GitHub strips CSS `style=` attributes entirely - and not only for PR Comments.

Even `<span style="color:yellow">` becomes plain text. And the basic `<font>` tags are rendered as literal text, not HTML.

The following both output as a plain `Yellow for prompts`
```html
<span style="color: Yellow;">Yellow</span> for prompts
<font color="Yellow">Yellow</font> for prompts
```

#### **LaTeX in GitHub Pages Markdown**

This block of LaTeX

```LaTeX
<summary>
   $\textcolor{green}{\textsf{suddenly}}$
   $\textcolor{yellow}{\textsf{ becomes}}$
   $\textcolor{red}{\textsf{ styled}}$
</summary>
```
Displays in GitHub Pages Markdown as plain text because its not rendered - and you get the following:
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
This is math‑mode LaTeX using `\color{red}` a math‑mode LaTeX colour switch. Nice colours, but that injects spaces before and after the verb noun hyphen (notOK). And you get that result when added to PR, README.md and here in GitHub Pages Markdown.

The format I blog about is different, this is text‑mode LaTeX using `\textcolor{red}` a text‑mode colour wrapper.
```
   $\textcolor{red}{\textsf{ styled}}$
```

---

These failures helped narrow down the little corner of GitHub's rendering engine where LaTeX color commands are interpreted.

---

## An Accidental Discovery: LaTeX Color Commands

I found some interesting references:
- [stackoverflow how-to-add-color](https://stackoverflow.com/questions/11509830/how-to-add-color-to-githubs-readme-md-file)  
- github markup issue [github/markup/issues/369](https://github.com/github/markup/issues/369)
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