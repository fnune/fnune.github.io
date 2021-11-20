---
layout: post
title: 'Making LunarVim work like my Neovim setup'
comments: true
date: 2021-11-20 21:35:51 +0200
excerpt: "Setting up Neovim to work like an IDE has become a maintenance burden. LunarVim looks like a promising solution that abstracts this away from me, so let's give it a try!"
---

The recent developments that landed with Neovim 0.5 provide it with built-in support for the [Language Server Protocol](https://neovim.io/doc/lsp/). Before [adopting this new feature](https://github.com/fnune/dotfiles/commit/64c128c8cebbf8296eb1326a034bc766ece27ae2) I had been using the wonderful [coc.nvim](https://github.com/neoclide/coc.nvim) for my language server needs, and although I was enjoying the new built-ins, the amount of work I needed to do to get stuff set up increased, and I had to get used to writing Lua.

My configuration became a Frankenstein of Vimscript and Lua, and this came with unexpected drawbacks such as having a syntax error in the middle of a Lua block be reported as if it had happened at the beginning of the block.

Setting up Neovim to work like an IDE was starting to become a maintenance burden for me, and I didn't have the patience to clean it all up again. Enter [LunarVim](https://www.lunarvim.org):

> LunarVim is an opinionated, extensible, and fast IDE layer for Neovim >= 0.5.0. LunarVim takes advantage of the latest Neovim features such as Treesitter and Language Server Protocol support.

That is exactly what I needed! In this post I'm going to cover my efforts in translating my existing, years-old Neovim configuration into a LunarVim configuration that behaves roughly the same. Some spoilers:

- My Neovim configuration wasn't as personal as I thought it was. As it turns out, vanilla LunarVim ticks almost all the boxes for me, and apparently so does it for over [six thousand users](https://github.com/LunarVim/LunarVim/stargazers) on GitHub. [My LunarVim configuration](https://github.com/fnune/dotfiles/blob/master/lvim/.config/lvim/config.lua) has 69 lines of Lua at the time of writing, whereas [my Neovim config](https://github.com/fnune/dotfiles/tree/master/neovim/.config/nvim) has 5 files and a whopping 439 lines of code.
- Adapting LunarVim to my needs has mostly been a breeze, but I did have to investigate a little bit more into a couple of issues.
- The UX inside of LunarVim is welcoming, discoverable and forgiving, so I would recommend it to Vim users who are just about done getting familiar with basic editing skills.
