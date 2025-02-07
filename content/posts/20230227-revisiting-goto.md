---
title: "Revisiting Goto"
date: 2023-02-27T13:55:48+08:00
---

Most developers have their preferred workflow. Mine is terminal-based which means I spend ~100% of my non-browser time on the terminal. Moving between folders was something I optimised in 2020 with my own "flavour" of a file-directory-changer.

I implemented https://github.com/slai11/goto instead of using off-the-shelf alternatives like jump or zoxide for 2 reasons: (1) those did not fit my usage pattern and (2) I wanted to build/hack together something in Rust.

goto served me well for 2 years. I used `gt` more than `cd` in my day-to-day workflow. However, I noticed that my use of `gt` fell after starting work at GitLab. Many project/repository titles have many common substrings: "gitlab". I ditched `gt` for a more accurate `cd` since false positives were high. This led me to wonder if I can just display my recently used folders and jump to one of them instead.

Enter the new mode: https://github.com/slai11/goto/pull/39. Obviously it needs more refinement. I think the list of files need to be chomped off after a certain length, else there is no point using `gt` if I'm spending 1-2 seconds scanning the list for my target file. These improvements will come when I've used it for a more extensive period of time and have a better idea of what I truly need.

In the meantime, I'm just happy to hack around in Rust. Being able to write in a modern programming language (verbose compiler feedback, dependency management, no manual memory management -- borrow checker could be less painful to use) while getting near C performance is truly a joy. Feels like writing Scala with zero cost abstractions and no JVM.
