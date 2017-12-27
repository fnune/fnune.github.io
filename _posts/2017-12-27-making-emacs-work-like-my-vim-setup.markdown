---
layout: post
title:  "Making Emacs work like my Neovim setup"
comments: true
date:   2017-12-27 12:47:51 +0200
---

## Table of contents

I've been a Neovim user and fan for a bit more than a year now. After having given it a reasonable spin I've become quite efficient at working with it, and it's been a pleasure all the way through. Certainly, I'm a lot faster with my Tmux/Neovim/gitsh workspace than I was with either Atom, Sublime Text or VSCode, and I feel a lot more comfortable.

> _Disclaimer: from this point and forward, and although I use Neovim, I'll be using the words Vim and Neovim interchangeably. Whether I refer to the software packages or to a specific user community should be clear in context._

During the last weeks I've noticed several tools and concepts in the Emacs which I've found attractive enough to try out the platform. These include:

- Org-mode: I've tried the Vim port and although it's a wonderful effort at emulating the original Emacs package, I think it would require quite a bit of an investment to reach the current scope of Org-mode. I plan to use Org-mode for GTD and for generic notetaking; also being able to write my Emacs configuration in Org-mode is a beauty.
- Magit: with my Tmux setup, I initialize several workspaces for each project with a script, and my standard workspace includes a Vim window, and another window with several panes. One of these is always a gitsh instance. It's worked wonderfully for me but after having tried the Magit interface there's no question that I'm going to be needing less keystrokes to do my thing, all while enjoying a beautiful interface.
- Lisp: admittedly, I could do with Vim but Emacs has a Lisp interpreter at its core, and integration is granted. I don't use Lisp at work and I'm a beginner, but it feels like it's impossible to find anything about Lisp support in Vim where the Emacs solutions are not mentioned.
- Integration: I like the _never leave your editor_ and _kitchen sink in Emacs_ approach and although I doubt I'll ever manage emails or browse the web inside Emacs, I feel all warm and fuzzy when I realize I could if I wanted to. Many of these things are arguably possible in Vim but it feels like the Emacs community leans more towards it than the Vim counterpart.

So I decided to surrender to my sacrilegous self and try to **emulate everything I do with Vim** from an empty Emacs config file built with Org-mode. And I must say: it's been a breeze! I haven't even needed to dedicate much time to learning actual Emacs, and what I've learned has actually been nice. In this post I'll try to go through what I did to rebuild my setup; I hope you'll enjoy it as much as I did.

## Quick reference table
    
### Package management

For package management needs the Vim community has contributed several awesome packages like [Pathogen]() or [vim-plug]() among the many worth mentioning. I've always used vim-plug and never found a problem with it. As active as the Emacs community is in regards to package development, I expected a solution that would provide the same level of comfort.

Emacs comes bundled with Package, and this is as much as I'm aware of: it takes care of package repository management, and to configure it I only needed to add the links to those repositories and initialize it.

Package, however, does not take responsibility for automatic fetching, updates, and encapsulation of configuration (which vim-plug does, and very well). For this, I've found the de-facto solution to be [use-package](). To be able to work with use-package using its minimal functionality, this is all you need to know:

- use-package can fetch whatever packages are made available through your Package configuration.
- A basic declaration looks like this: `(use-package package-name)`.
- If you add `:ensure t`, you'll get automatic fetching of your package and startup checks: `(use-package package-name :ensure t)`.
- If you add `:defer t`, your package will load lazily: `(use-package package-name :ensure t :defer t)`.
- You can add `:init`, and everything you pass it will be evaluated _before_ the package loads. Here's where you'll use `(setq key 'value)`, for example.
- You can add `:config`, and everything you pass it will be evaluated _after_ the package loads. Here's where you'll initialize modes, for example.

It didn't take me too long to learn this, and use-package allegedly does a thousand more things which I'll begin to learn with time.

### Vim things and Evil things

[Evil](https://github.com/emacs-evil/evil) calls itself the _extensible vi layer for Emacs_, and claims that it _emulates the main features of Vim_. I'd say this is an understatement; Evil feels like a complete re-implementation of Vim. It makes you feel right at home once you start using it:

- Macros: these work exactly as expected. Even making a visual selection and running `:norm @q` runs your `q` macro on the visual selection, just like in Vim. The only difference I've noticed is that execution is minimally slower, but the decrease in speed does not compare to that of VSCode's implementation of Vim macros, for example.
- Registers: these also work exactly as expected. The only problem I've had is that I can't copy to the clipboard by using the `+` register, but this must be a misconfiguration on my part for Emac's clipboard integration, so I suspect it won't be a huge effort to fix it.
- Command repetition (`.`): works as expected, except for some actions introduced by other packages. One of these, unfortunately, is [evil-surround](https://github.com/emacs-evil/evil-surround). [Here's the related issue](https://github.com/emacs-evil/evil-surround/issues/133).
- Auto-save and safety/backup features: these can be easily configured to not happen at all or to happen in a specified directory (I'm using `/tmp`).
- Ex commands (those starting with a colon `:`) like substitution, substitution with manual confirmation, invocation of macros in normal mode, etc. All work great and I haven't found an instance where they don't.
- Marks: I don't make extensive use of these, but they also seem to be working great.

Using [evil-leader]() you can configure a leader key. I've configured mine to `Space`, and added a several keybindings. The same results can be achieved with the more powerful [general.el](), and if you need chained keystrokes to produce a command (for example, I used to have `<leader> wq`, which I found faster than `:wq`), you can use [Hydra](). I haven't found a need for these and I'm doing just fine with evil-leader.

### Project management and file navigation

My setup using Vim is basically [fzf]() (which I use for many more things outside Vim) powered by [Ag (or The Silver Searcher)]() for finding files and [ripgrep]() for finding text in a project. This works flawlessly.

I've found the combination of [Helm]() and [Projectile]() to be an adequate substitute to my former setup. On big projects like Servo, the difference in speed is noticeable (in favor of the Vim configuration) but I can live with that. I don't know why, but there's a longer load time on the Emacs setup.

The scope of fzf is by no means comparable to that of Helm and Projectile, so this is not meant to be a comparison but it does happen to be what covers my file-finding needs. Both setups enable extremely quick fuzzy search for files and content.

As you can see [on my Emacs configuration](), my setup for Helm and Projectile is extremely basic and I haven't needed further customization yet. And I must say: they look much prettier than the Vim setup I use.

### Specific packages

A quick search on your favorite engine will yield at least a couple different solutions to problems some of the nicest Vim plugins solve. Here's a quick list to encourage you:

- [VimCompletesMe](): I enjoyed the simplicity of VimCompletesMe, which basically only extends Vim's autocomplete features and lets you use them by pressing `Tab`. I found that the Emac package [auto-complete]() provides the same ease of use and also feels lightweight.
- [vim-tmux-navigator](): in Tmux, I use `<my-tmux-prefix>-[hjkl]` to navigate panes. Using Vim, I wanted windows to behave as if they were on the same level as Tmux panes, and vim-tmux-navigator works great for that. For Emacs there's a port called [emacs-navigator]().
- [auto-pairs](): Emacs has a built-in mode that suits my needs. Enable it with `(electric-pair-mode 1)`.
- [NerdTree](): the Emacs port [NeoTree]() does the original justice and, although I haven't gotten there yet, it can also be extended with Git integration and icons if you use GUI Emacs.
- [vim-emoji-complete](): I use this to navigate and autocomplete through a list of Unicode emojis. In the company I work at, we use [Gitmojis]() extensively, so this is actually an important part of my workflow. You should check them out too, it may seem silly but it's quite helpful to be able to recognize what every commit does without even reading the message. For Emacs, there's an even better solution for inserting emojis into your buffer: [emojify](). This thing even lets you customize the list of emojis you get. For example, I've chosen to only display Unicode emojis, and not GitHub or vanilla ASCII emojis.

Regarding [Tim Pope plugins](https://github.com/tpope?tab=repositories): there's an Emacs port for everything Mr. Pope does. Many of these go on top of Evil, and it's a no-brainer to add them and use them if you're used to their Vim counterpart.

### Theming

### Performance and server mode

## Conclusion
