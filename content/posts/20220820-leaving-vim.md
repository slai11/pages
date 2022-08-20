---
title: "Leaving Vim"
date: 2022-08-20T09:20:09+08:00
tags:
- fragments
---
## The betrayal

GitLab monorepo is way too large to use with my existing neovim setup. Even with udpated solargraph config to index 20000 files, using fzf+ripgrep.

The way ruby is written also involves patching and overriding gem functions/methods. After much frustration in trying to navigate the monstrous monorepo, I gave up and tried Rubymine. John Carmack's interview on [YouTube](https://www.youtube.com/watch?v=tzr7hRXcwkw) helped me feel less guilty about ditching vim. tldr; use the best tools for the job, dont be too constrained by the editor war/cultism.

JetBrain's Rubymine has a really impressive jump-to-definition and reference. I can jump to the gem's definition which was extremely helpful in working on Sidekiq tasks.

I lost the speed and customizability of neovim, my ripgrep-fzf powered search (which was quite slow in a monorepo despite a 32GB RAM and M1 Max processor), my multitudes of shortcut to navigate through a file, and many more. But in return, cmd+B always gives me the right definition. I suspect the debugger is powerful but I have yet to confirm that.


## The saving grace

That said, neovim is still my go-to editor for everything else, Go, Rust, yamls, bash, etc. The lightweight ability to type `nvim xxx` to open a file then fly through it with quick edits is still unrivalled. 

After 3 years of using neovim, I doubt I can fully abandon it. If anything, Rubymine is the exception not the norm. I hope that when the time comes for me to work on Go projects in GitLab, I do not ditch neovim for Goland.

## The dream

The ultimate dream is neovim (or some new iteration of vim) and the ecosystem LSP libraries being able to maximise the power of modern processors and rival the experience of JetBrain IDEs. Til then, I will continue my work in GitLab using Rubymine.
