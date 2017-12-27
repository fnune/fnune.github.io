---
layout: post
title:  "Making Emacs work like my Neovim setup"
comments: true
date:   2017-12-27 12:47:51 +0200
---

## Table of contents

# Making Emacs work like my Neovim setup

I've been a Neovim user and fan for a bit more than a year now. After having given it a reasonable spin I've become quite efficient at working with it, and it's been a pleasure all the way through. Certainly, I'm a lot faster with my Tmux/Neovim/gitsh workspace than I was with either Atom, Sublime Text or VSCode, and I feel a lot more comfortable.

_**Disclaimer**: from this point and forward, and although I use Neovim, I'll be using the words Vim and Neovim interchangeably. Whether I refer to the software packages or to a specific user community should be clear in context._

During the last weeks I've noticed several tools and concepts in the Emacs which I've found attractive enough to try out the platform. These include:

- Org-mode: I've tried the Vim port and although it's a wonderful effort at emulating the original Emacs package, I think it would require quite a bit of an investment to reach the current scope of Org-mode. I plan to use Org-mode for GTD and for generic notetaking; also being able to write my Emacs configuration in Org-mode is a beauty.
- Magit: with my Tmux setup, I initialize several workspaces for each project with a script, and my standard workspace includes a Vim window, and another window with several panes. One of these is always a `gitsh` instance. It's worked wonderfully for me but after having tried the Magit interface there's no question that I'm going to be needing less keystrokes to do my thing, all while enjoying a beautiful interface.
- Lisp: admittedly, I could do with Vim but Emacs has a Lisp interpreter at its core, and integration is granted. I don't use Lisp at work and I'm a beginner, but it feels like it's impossible to find anything about Lisp support in Vim where the Emacs solutions are not mentioned.
- Integration: I like the _never leave your editor_ and _kitchen sink in Emacs_ approach and although I doubt I'll ever manage emails or browse the web inside Emacs, I feel all warm and fuzzy when I realize I could if I wanted to. Many of these things are arguably possible in Vim but it feels like the Emacs community leans more towards it than the Vim counterpart.

So I decided to surrender to my sacrilegous self and try to **emulate everything I do with Vim** from an empty Emacs config file built with Org-mode. And I must say: it's been a breeze! I haven't even needed to dedicate much time to learning actual Emacs, and what I've learned has actually been nice. In this post I'll try to go through what I did to rebuild my setup; I hope you'll enjoy it as much as I did.

## Quick reference table
    
### Package management

For package management needs the Vim community has contributed several awesome packages like [Pathogen]() or [vim-plug]() among the many worth mentioning. I've always used `vim-plug` and never found a problem with it. As active as the Emacs community is in regards to package development, I expected a solution that would provide the same level of comfort.

Emacs comes bundled with Package, and this is as much as I'm aware of: it takes care of package repository management, and to configure it I only needed to add the links to those repositories and initialize it.

Package, however, does not take responsibility for automatic fetching, updates, and encapsulation of configuration (which `vim-plug` does, and very well). For this, I've found the de-facto solution to be [use-package](). To be able to work with `use-package` using its minimal functionality, this is all you need to know:

- `use-package` can fetch whatever packages are made available through your Package configuration.
- A basic declaration looks like this: `(use-package package-name)`.
- If you add `:ensure t`, you'll get automatic fetching of your package and startup checks: `(use-package package-name :ensure t)`.
- If you add `:defer t`, your package will load lazily: `(use-package package-name :ensure t :defer t)`.
- You can add `:init`, and everything you pass it will be evaluated _before_ the package loads. Here's where you'll use `(setq key 'value)`, for example.
- You can add `:config`, and everything you pass it will be evaluated _after_ the package loads. Here's where you'll initialize modes, for example.

It didn't take me too long to learn this, and `use-package` allegedly does a thousand more things which I'll begin to learn with time.

### Sane defaults and basic Emacs settings

### Vim things and Evil things
    
- Macros
- Registers
- Command repetition (`.`)
- Auto-save and safety/backup features

### Keybindings and the leader key

### Project management and file navigation

### Git workflow

### Linting

### Specific packages

- VimCompletesMe
- vim-tmux-navigator
- auto-pairs
- NerdTree
- vim-emoji-complete
- Tim Pope's stuff

### Theming

### Splits and Tmux integration

### Language-specific support

### Performance and server mode

## Conclusion
