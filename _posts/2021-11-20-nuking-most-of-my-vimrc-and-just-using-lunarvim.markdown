---
layout: post
title: 'Nuking most of my .vimrc and just using LunarVim'
comments: true
date: 2021-11-20 21:35:51 +0200
excerpt: "Setting up Neovim to work like an IDE has become a maintenance burden. LunarVim looks like a promising solution that abstracts this away from me, so let's give it a try!"
---

The recent developments that landed with Neovim 0.5 provide it with built-in support for the [Language Server Protocol](https://neovim.io/doc/lsp/). Before [adopting this new feature](https://github.com/fnune/dotfiles/commit/64c128c8cebbf8296eb1326a034bc766ece27ae2) I had been using the wonderful [coc.nvim](https://github.com/neoclide/coc.nvim) for my language server needs. With all the new tricks Neovim was learning, the amount of work I needed to do to get everything working the way I wanted increased steadily.

My configuration became a Frankenstein of Lua and Vimscript, and this came with unexpected drawbacks such as having a syntax error in the middle of a Lua block be reported as if it had happened at the beginning of the block.

Setting up Neovim to work like an IDE was starting to become a maintenance burden for me, and I didn't have the patience to clean it all up again. Enter [LunarVim](https://www.lunarvim.org):

> LunarVim is an opinionated, extensible, and fast IDE layer for Neovim >= 0.5.0. LunarVim takes advantage of the latest Neovim features such as Treesitter and Language Server Protocol support.

That is exactly what I needed! In this post, I'm going to cover my efforts in translating my existing, years-old Neovim configuration into a LunarVim configuration that behaves roughly the same. Some spoilers:

- My Neovim configuration wasn't as personal as I thought it was. As it turns out, vanilla LunarVim ticks almost all the boxes for me, and it appears so does it for over [six thousand users](https://github.com/LunarVim/LunarVim/stargazers) on GitHub. [My LunarVim configuration](https://github.com/fnune/dotfiles/blob/master/lvim/.config/lvim/config.lua) has 69 lines of Lua at the time of writing, whereas [my Neovim config](https://github.com/fnune/dotfiles/tree/master/neovim/.config/nvim) has 5 files and a whopping 439 lines of code.
- Adapting LunarVim to my needs has mostly been a breeze, but I did have to investigate a little bit more into a couple of issues.
- The UX inside of LunarVim is welcoming, discoverable and forgiving, so I would recommend it to Vim users who are just about done getting familiar with basic editing skills.

## Keybindings

Most of [my mappings](https://github.com/fnune/dotfiles/blob/master/neovim/.config/nvim/mappings.vim) ended up being redundant.

LunarVim makes excellent use of [`which-key`](https://github.com/folke/which-key.nvim), a pop-up window that suggests options to help you complete a keybinding.

![The which-key plugin in LunarVim after pressing the leader key](/img/lvim/which-key-leader.png)
_The which-key plugin in LunarVim after pressing the leader key_

There's the reassuring certainty that I'll be able to find the keybinding again if I've found it once, by exploring the nested menus that update as you select an option.

Once I've learned a combination, however, it quickly becomes second nature: the keys are simply well-chosen. I remembered my first experience with [Magit](https://magit.vc/) as I found that the combination to _search_ for _text_ is `st`, and as I realized that this bit of knowledge would allow me to discover other things that are searchable, by just pressing `s` and waiting `timeoutlen` milliseconds for the `which-key` pop-up to appear.

I did end up adding exactly four keybindings that I missed:

```lua
vim.cmd("map 0 ^")
```

![map 0 ^](/img/lvim/zero.png)
_I rarely want my cursor to go to position (2). By default, `0` takes you to the first column, but I want it to take me to the first character in the line._

```lua
vim.cmd("nnoremap Q <nop>")
```

![nnoremap Q nop](/img/lvim/ex.png)
_I [never became friends with Ex mode](https://vi.stackexchange.com/questions/457/does-ex-mode-have-any-practical-use), and seeing this always frustrated me, so I disabled its keybinding._

```lua
vim.cmd("nnoremap j gj")
vim.cmd("nnoremap k gk")
```

![nnoremap j gj](/img/lvim/gj.png)
_If you write prose and you've enabled line-wrapping, then remapping `j` to `gj` and `k` to `gk` is a great idea. Pressing `j` here will take me to position (1) and not (2)._

## Fuzzy finder

I've used the magnificent [`fzf.vim`](https://github.com/junegunn/fzf.vim) for some years now. LunarVim comes with [Telescope](https://github.com/nvim-telescope/telescope.nvim), though, and there are some things I'm enjoying about it.

One of them is that it simply looks great: it goes really well together with a patched font (in this case `Cascadia Code SemiLight`) thanks to the iconography, and the preview on the right is syntax-highlighted, and it'll hide responsively depending on the available size of the terminal.

![Telescope in LunarVim](/img/lvim/telescope.png)
_It just looks beautiful_

Each fragment in a file path can be truncated to fit a longer path in the screen. I thought this would be confusing at first, but I've had no trouble with it.

I have no complaints about the file fuzzy finder, but the "live grep" functionality has one big drawback. With `fzf.vim`, I used to be able to do this:

```vim
:Rg search_string
```

This would pop up a view similar to Telescope, and I could fuzzy-filter _the search results_. This was incredibly useful, for example, if I want to find `search_string` but realize after the fact that most results are in Python files, while I'm interested in the TypeScript results. I would then simply type `.ts` and get all files containing `search_string` in TypeScript files only. Very flexible!

Unfortunately, this is not possible in vanilla Telescope, and I've yet to find a solution for this. It is a big part of my daily workflow, so I might revert to `fzf.vim` for my text-search needs.

<div id="grep-input-string" />

_Thanks to [two helpful Redditors](https://old.reddit.com/r/neovim/comments/qysjgf/nuking_most_of_my_vimrc_and_just_using_lunarvim/hlkxhjl/), I've managed to find a solution for this. With the following snippet, pressing `<leader>sT` will do something very similar to `:Rg`, and will also take the word under the cursor as an initial value:_

```lua
function GrepInputString()
  local default = vim.api.nvim_eval([[expand("<cword>")]])
  local input = vim.fn.input({
    prompt = "Search for: ",
    default = default,
  })
  require("telescope.builtin").grep_string({ search = input })
end
lvim.builtin.which_key.mappings["sT"] = { "<cmd>lua GrepInputString()<CR>", "Text under cursor" }
```

## File explorer

I was a happy user of [`nnn`](https://github.com/jarun/nnn), a terminal file explorer with [a Vim plugin](https://github.com/mcchrish/nnn.vim). It's great, but some features are compile-in, which may be a nuisance to some.

LunarVim comes with [`nvim-tree`](https://github.com/kyazdani42/nvim-tree.lua). Honestly, I can't complain. I wasn't an advanced user of `nnn`. In `nvim-tree`, all features are quickly discoverable by entering `g?` ("go to help", in and of itself an easily discoverable combination).

![nvim-tree in LunarVim](/img/lvim/nvim-tree.png)
_The file explorer opens with the file from the current buffer selected. It also (in my opinion) benefits greatly from the patched font._

## Git interface

During [my time using Emacs](/2017/12/27/making-emacs-work-like-my-vim-setup) I became a fan of [Magit](https://magit.vc/). It's a great piece of software that changed my expectations of what a Git interface can do.

Since then, I've been through different solutions, including using an always-running [Emacs instance in server mode](https://www.gnu.org/software/emacs/manual/html_node/emacs/Emacs-Server.html) running only Magit, which I accessed inside of Neovim and other programs. Later on, I used [tpope's `vim-fugitive`](https://github.com/tpope/vim-fugitive) and tried to adapt [some Magit-inspired combinations](https://github.com/fnune/dotfiles/blob/master/neovim/.config/nvim/mappings.vim#L74-L84) such as `p-fp` for "push", "force", "push".

LunarVim's porcelain of choice is [`lazygit`](https://github.com/jesseduffield/lazygit). I'd been meaning to try `lazygit` for a while now, and this was the perfect opportunity. The "unboxing" experience was great: `lazygit` shows a link to [a YouTube video](https://www.youtube.com/watch?v=CPLdltN7wgE) by the author Jesse Duffield. I can't make the software justice in this introduction, so I would recommend that you simply try it out.

![lazygit in LunarVim](/img/lvim/lazygit.png)
_Obligatory screenshot. Please [watch the YouTube video](https://www.youtube.com/watch?v=CPLdltN7wgE) for a proper introduction._

One minor drawback of using `lazygit` inside of Neovim is that its `open` and `edit` commands when pointing at files will open a nested `$EDITOR` inside of the `lazygit` terminal. With `vim-fugitive`, opening a file in its UI would simply reuse the current Neovim instance. I had this expectation of `lazygit` running inside LunarVim as well. I've read about using [Neovim in remote mode](https://github.com/mhinz/neovim-remote) to solve this, but I haven't gotten around to it. I do feel like if LunarVim picks `lazygit` as its integrated Git interface, it could also come with a working solution for this.

## Verdict

I think I'll keep using LunarVim. With my old configuration, It's a rare week if I don't need to fix one or two things on one of my LSP setups, or any of the IDE-like features that Neovim plugins offer. This all just works mostly out of the box on LunarVim, and not having to maintain this is a great relief. I'm looking forward to not being in complete control of this.

If you can relate, I'd like to invite you to give [LunarVim](https://www.lunarvim.org) a try and see if you like it. Thanks for reading my blog!
