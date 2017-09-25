---
title:  "Show Git Status in ZSH Prompt with Respect to a Remote Repo"
---

*zsh* is a powerful shell that allows you to show your current *git* status in your command prompt: [Show git state in zsh prompt via vcs_info]({% post_url 2016-04-23-zsh-prompt-and-vcs_info %}). It supports current branch, repo path and many other options, most notably *stage/unstaged* symbols to show you current git dirty state.

Here is how to reconfigure *zsh* to respect remote repo e.g. *GitHub*'s status properly.

<!--more-->

## Result

Here is an end result, check below for implementation details.

```sh
zstyle ':vcs_info:git+post-backend:*' hooks git-remote-staged
function +vi-git-remote-staged() {
  # Show "unstaged" when changes are not staged or not committed
  # Show "staged" when last committed is not pushed
  #
  # See original VCS_INFO_get_data_git for implementation details

  # Set "unstaged" when git reports either staged or unstaged changes
  if (( gitstaged || gitunstaged )) ; then
    gitunstaged=1
  fi

  # Set "staged" when current HEAD is not present in the remote branch
  if (( querystaged )) && \
     [[ "$(${vcs_comm[cmd]} rev-parse --is-inside-work-tree 2> /dev/null)" == 'true' ]] ; then
      # Default: off - these are potentially expensive on big repositories
      if ${vcs_comm[cmd]} rev-parse --quiet --verify HEAD &> /dev/null ; then
          gitstaged=1
          if ${vcs_comm[cmd]} branch -r --contains 2> /dev/null | read REPLY ; then
            gitstaged=
          fi
      fi
  fi

  hook_com[staged]=$gitstaged
  hook_com[unstaged]=$gitunstaged
}
```

## *Staged/unstaged* state

Here is how *staged/unstaged* marks works out of the box:

  - *unstaged*: you have changes that you have not staged (added) to your git repo
  - *staged*: you have changes that you staged (added) to your current git repo, but you have not created a commit yet

In my workflow there is not a big difference between staged and not yet staged files is not important and this indicator is not very useful. But it's *very* important if I had pushed my changes to the remote repo (such as *GitHub*) or not.

Here is how *staged/unstaged* marks should work:

  - *unstaged*: you have changes that you have not committed (either staged or not)
  - *staged*: you committed some changes that were not pushed yet

## *zsh* implementation

*zsh* already has *vcs_info* hook: `/usr/share/zsh/$ZSH_VERSION/functions/VCS_INFO_get_data_git`, which I use as a base for my implementation.

### First, you need to add your own hook to *zsh*

```sh
## ~/.zshrc
zstyle ':vcs_info:git+post-backend:*' hooks git-remote-staged
function +vi-git-remote-staged() {
  # .... set $gitstaged and $gitunstaged

  hook_com[staged]=$gitstaged
  hook_com[unstaged]=$gitunstaged
}
```

### Show *unstaged* when git reports either *staged* or *unstaged* changes

Environment variables created by *zsh* original *git* integration are present and we can check against them directly:

```sh
## Set "unstaged" when git reports either staged or unstaged changes
if (( gitstaged || gitunstaged )) ; then
  gitunstaged=1
fi
```

### Show *staged* when committed change is not in the remote branch

It was important for me to make sure that my code works well when remote repository is not available. Any kind of `merge-base` or other complicated commands require me to call `fetch` to show up-to-date information. It's not feasible for command prompt rendering as it has to be very fast.

And the only check that does not change its value when remote repo is not available is a presence of a current commit in the remote branch. Local client always knows if it pushed this commit or not and never needs to consult remote repository. Even better, calling `git fetch` would not change results of this check.

Find all remote branches containing current commit:

```sh
git branch -r --contains
```

Unfortunately it always has `0` return status, so I'm using `read` to detect if no branches are found.

> I'm writing code with respect to the original hook codebase, it may be more verbose than necessary, but will be more portable.

```sh
## Set "staged" when current HEAD is not present in the remote branch
if ${vcs_comm[cmd]} rev-parse --quiet --verify HEAD &> /dev/null ; then
    gitstaged=1
    if ${vcs_comm[cmd]} branch -r --contains 2> /dev/null | read REPLY ; then
      gitstaged=
    fi
fi
```

And that's basically it!

### Full text of my current `.zshrc`

<script src="https://gist.github.com/timothybasanov/87df55aad8ca8afe40d2.js"></script>
