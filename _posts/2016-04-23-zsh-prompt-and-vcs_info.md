---
layout: post
title:  "Show Git State in ZSH Prompt via vcs_info"
date:   2016-04-23
---

<!-- Some magic from TextEdit -->
  <style type="text/css">
    p.p1 {margin: 0.0px 0.0px 0.0px 0.0px; line-height: 25.0px; font: 22.0px Menlo; color: #ededed; -webkit-text-stroke: #000000; background-color: #000000}
    p.p2 {margin: 0.0px 0.0px 0.0px 0.0px; line-height: 25.0px; font: 22.0px Menlo; color: #ededed; -webkit-text-stroke: #000000}
    p.p3 {margin: 0.0px 0.0px 0.0px 0.0px; line-height: 25.0px; font: 22.0px Menlo; color: #36be29; -webkit-text-stroke: #000000}
    span.s1 {font-kerning: none}
    span.s2 {font-kerning: none; background-color: #000000}
    span.s3 {font-kerning: none; color: #ededed; background-color: #000000}
    span.s4 {font-kerning: none; color: #c63a23; background-color: #000000}
    span.s5 {font-kerning: none; color: #35bdc9; background-color: #000000}
    span.s6 {font-kerning: none; color: #ff3e1f; background-color: #000000}
    span.s7 {font: 22.0px 'Apple Symbols'; font-kerning: none; color: #ffffff; background-color: #000000}
  </style>
<!-- Making sure of a monospace font -->
<style type="text/css">
  p.p1, p.p2, p.p3 {  font-family: "Lucida Console", Monaco, monospace }
</style>

Zsh has excellent capabilities of supporting different vcs, like git. There is plenty of [generic prompt documentation](http://zsh.sourceforge.net/Doc/Release/Prompt-Expansion.html) and [vcs-specific prompt documentation](http://zsh.sourceforge.net/Doc/Release/User-Contributions.html#Version-Control-Information) and it's very-very long. I'm trying to make it as short as possible and as simple as possible without loosing all the important points when you'll want to customize it for yourself.

I'll show evolution of a command line prompt after each modification. I'll assume `~/.git` exists and has changes on the master branch. Current directory is set to `~/.ssh`.

All the changes mentioned are done in `~/.zshrc` file.

### Show current user and directory in the command prompt

<p class="p1"><span class="s1">timothy@home:~/.ssh&gt;<span class="Apple-converted-space"> </span></span></p>

Zsh supports username `%n`, short hostname `%m` and directory `%~` in prompt  (there are [many more available](http://zsh.sourceforge.net/Doc/Release/Prompt-Expansion.html)):

 - Let's show `user@host:dir> ` prompt: `PROMPT='%n@%m:%~> '`

It's pretty basic, but that's only the beginning.

### Show git (or any other version management system) info in the prompt

<p class="p1"><span class="s1">timothy@home: (git)-[master]-&gt;<span class="Apple-converted-space"> </span></span></p>

  - Allow dynamic command prompt: `setopt prompt_subst`
  - Make sure that `vcs_info` function is available: `autoload -Uz vcs_info`
  - Update each time new prompt is rendered: `function precmd() { vcs_info }`
  - Show git info in the prompt: `PROMPT='%n@%m:${vcs_info_msg_0_}> '`

Now we can see that we're within git repository.

### Add repository path and path within repository to the prompt

<p class="p1"><span class="s1">timothy@home:/Users/timothy/(master).ssh&gt;<span class="Apple-converted-space"> </span></span></p>

  - Show full path when outside of git: `zstyle ':vcs_info:*' nvcsformats '%~'`
  - Show base (.git) directory, current branch and path within a repo: 
    `zstyle ':vcs_info:*' formats '%R/(%b)%S'`

That's much better, we got back information about our current directory

### Add information about uncommitted changes

<p class="p1"><span class="s1">timothy@home:/Users/timothy/(master).ssh!&gt;<span class="Apple-converted-space"> </span></span></p>

  - Show "unsubmitted changes" mark: `zstyle ':vcs_info:*' check-for-changes true`
  - Add `!` for unstaged changes: `zstyle ':vcs_info:*' unstagedstr '!'`
  - Add `+` for staged changes: `zstyle ':vcs_info:*' stagedstr '+'`
  - Render both marks if they are present: `zstyle ':vcs_info:*' formats '%R/(%b)%S`**`%u%c'`**

Good, we see `!` as we have uncommitted changes, as expected.

### Supporting merge and other unusual stages

Sometimes "additional information" `%m` is provided, better to have it than not:

  - Miscellaneous data support: `zstyle ':vcs_info:*' formats '%R/(%b)%S%u%c`**`%m'`**

When git is in a special state, e.g. merge, branch `%b` information should be replaced with "current state" `%a` information. Also it's a separate configuration string, `actionformats` instead of familiar `formats` one.

  - Support special (merge) state: `zstyle ':vcs_info:*' `**`actionformats`**` '%R/(`**`%a`**`)%S%u%c%m'`

### Use `~` for the home directory once again

<p class="p1"><span class="s1">timothy@home:~/(master).ssh!&gt;<span class="Apple-converted-space"> </span></span></p>

When we switched from zsh's default `%~` to git-specific `%R` we lost support for showing `~` instead of a home directory, let's get it back!

```sh
zstyle ':vcs_info:*+set-message:*' hooks home-path
function +vi-home-path() {
  autoload -U regexp-replace
  hook_com[base]="${hook_com[base]/$HOME/~}"
}
```

# Making it look nicer!

<p class="p3"><span class="s2">timothy</span><span class="s3">@</span><span class="s4">home</span><span class="s3">:</span><span class="s5">~/</span><span class="s2">(master)</span><span class="s5">.ssh</span><span class="s6"><b>●</b></span><span class="s7">⟫</span><span class="s3"><span class="Apple-converted-space"> </span></span></p>

Zsh supports colors, of course, but they are very verbose: `%{``%F{red}``%} red text %{``%f``%}`. Other colors: `black`, `red`, `green`, `yellow`, `blue`, `magenta`, `cyan` and `white`. Bold text is also supported: `%{``%B``%} bold text %{``%b``%}`. It's recommended to use `%{``%E``%}` to reset colors right before the end of your prompt.

And, almost certainly UTF-8 is supported on your terminal so feel free to use Unicode symbols as well.

  - Example of `stagedstr` after making it nicer: `zstyle ':vcs_info:*' stagedstr '%{``%F{green}%B%}`**`●`**`%{``%b%f%}`.

For the full source code see code snippets at the end of this post.

## Full text of my current `.zshenv` and `.zshrc`

<script src="https://gist.github.com/timothybasanov/87df55aad8ca8afe40d2.js"></script>

