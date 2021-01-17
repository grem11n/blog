---
title: "NeoVim GUI configuration"
description: "Creating a NeoVim setup with neovim-qt on macOS"
date: 2021-01-17
draft: false
slug: "nvim-en"
image: "https://cdn.dribbble.com/users/2008/screenshots/1442436/mark-dribbble_1x.png"
categories: ["Tech"]
tags: ["nvim", "neovim", "vim", "eng"]
---

{{< figure src="/img/posts/nvim/nvim-qt.png" >}}

I'm writing a post on NeoVim and NeoVim GUI configuration mostly for myself. Even though I have [all my dotfiles on the Github](https://github.com/grem11n/dotfiles), some configuration requires gluing. Also, some things have to be executed in a particular order and some configs must exist in particular places. I spent some time looking for fixes for occasional issues and polishing some rough edges. So, here I just want to put it all together, so I won't need to go down the googling spiral if I need to set up a new development environment for myself.

Since I've decided to keep some of my notes open, I need to provide a bit of context for those, who might be reading it. Therefore, this text comes in the shape of a blog post and not just a bullet-list. I should probably write something like an Ansible playbook to automate all these things. I'm just too lazy to do so. Also, this article is mostly focused on the GUI part, because a plain configuration of NeoVim is pretty straightforward.
A bit of history first. I've been using NeoVim for a while. Initially, I switched because NeoVim provided async APIs, while Vim prior version 8 did not. I've been using NeoVim since then for my day-to-day work. Also, I'm using [NeoVim native language protocol](https://github.com/neovim/nvim-lspconfig) implementation, which is another point for this editor.

Now the GUI part. I find it very powerful to be able to run your development environment inside a terminal application. I use [iTerm2](https://iterm2.com/) daily. Actually, this is one of my always-on applications on the work laptop. Hence, I didn't pay much attention to what's going on in that field. Things changed when I bought myself an ultrawide monitor. See, the biggest advantage of a UW-monitor is that you can keep all required applications on a display. For me, these apps are aforementioned iTerm2, a Web browser, and Slack. I also have about 1/6 of the display space vacant for other things like a Calendar, YouTube videos, etc.

{{< figure src="/img/posts/nvim/wide-screen.png" >}}

This layout works very well for me, but it comes with a notable disadvantage: You have only 1/3 of the display space for your terminal, aligned vertically. This is completely fine for regular terminal usage, but maybe cumbersome for using a text editor. Even "not-that-long" lines can be broken into two lines and any built-in file manager becomes effectively useless. I want to occasionally have a wider space to write/check the source code, but I don't want to resize my iTerm2 app all the time.

Why not use an IDE? A few reasons: in most cases, I don't need the whole power of an IDE. Second, I value consistency. I can SSH to almost any machine and it will have vi or at least installed. Third, muscle memory. For many years [Neo]Vim as my primary editor and I'm just getting super used to its commands and key bindings. I just don't see the point of learning this stuff once again. Of course, almost every IDE nowadays has some kind of a Vim-keybindings plugin, but the previous two arguments are still valid.

This was the time to look at NeoVim GUIs once again. There are [quite a few implementations](https://github.com/neovim/neovim/wiki/Related-projects#gui), actually. Some of them are inactive or abandoned. Some others are written on top of the Electron framework (I am biased here). Others have random glitches on macOS. To be honest, the only GUI that worked for me is [neovim-qt](https://github.com/equalsraf/neovim-qt). It has some problems out of the box, but I managed to fix them. And now I am writing this article in order to not forget how.

## Basic NVim installation

I use a "non-stable" 0.5 branch of NeoVim, which can be installed on macOS with brew:

```bash
brew install neovim --HEAD
```

And updated with

```bash
brew upgrade neovim --fetch-HEAD
```

For some reason, command-line arguments to fetch the latest code are different for `install` and `upgrade` operations.

Now I can just put my configs from [dotfiles](https://github.com/grem11n/dotfiles) to the respective directories. I tried to keep a filesystem structure as much as possible in this repo. Therefore, you can just add a dot (`.`) in front of level 1 files and directories. All my configuration is in the `.vimrc`file. Sometimes, I use "versioning" for them inside the repo. It usually happens when I refactor something there and don't want to get rid of the old stuff while evaluating the new one.

I use [vim-plug](https://github.com/junegunn/vim-plug) as my plugin manager. It should be installed before aeverything else. Just follow the installations directives from README. Once it's installed, all other plugins can be installed by calling a Vim command:

```vim
:PlugInstall
```

Now, once all the things are in place, it worths running NeoVim healschedk:

```vim
:checkhealth
```

Very likely there will be some errors, which I'd rather fix before jumping into the GUI part.

## GUI: NeoVim-QT

[NeoVim-QT](https://github.com/equalsraf/neovim-qt) is written in C++ and should be compatible with all the platforms. This is the only GUI implementation, which works well for me at MacOS. It does have issues out of the box, though. 

### GUI-specific config

GUI requires some specific configuration. Moreover, some of the required parameters don't work well with nvim in a terminal, mouse config, for example. Luckily, NeoVim can separate GUI configuration from the generic one. Just put the required parameters into `$HOME/.config/nvim/ginit.vim`. For example:

```vim

" Required for Neovim-qt GUI
set guifont=Hack\ Nerd\ Font:h16
set mouse=a
so $HOME/.config/nvim/macmap.vim
set path+=/Users/yurii.rochniak/.nvm/versions/node/v14.15.0/bin:/usr/local/Cellar/tfenv/1.0.1/bin:/Users/yurii.rochniak/.yarn/bin:/Users/yurii.rochniak/.config/yarn/global/node_modules/.bin:/Users/yurii.rochniak/.krew/bin:/Users/yurii.rochniak/Golang/bin:/Users/yurii.rochniak/Library/Python/3.8/bin:/Users/yurii.rochniak/Library/Python/2.7/bin:/usr/local/sbin:/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin:/opt/X11/bin:/usr/local/git/bin:/usr/local/MacGPG2/bin:/usr/local/go/bin:/Users/yurii.rochniak/.cargo/bin

let $PERL5LIB = '/Users/yurii.rochniak/perl5/lib/perl5'
let $PERL_LOCAL_LIB_ROOT = '/Users/yurii.rochniak/perl5'
let $PERL_MB_OPT = '--install_base "/Users/yurii.rochniak/perl5"'
let $PERL_MM_OPT = 'INSTALL_BASE=/Users/yurii.rochniak/perl5'
```



### macOS key mapping

Notice line 4 of the config above. I'm sourcing another .vimfile. This file contains some important key mappings for MacOS. Like mappings, which allow you to copy-paste things in and from your system buffer with CMD+C & CMD+V. These mappings are imported automatically when you use NeoVim in your terminal app, but NeoVim-QT doesn't load that by default. You can find the original file in `<PATH_TO_YOUR_NVIM_INSTALLATION>/share/nvim/runtime/macmap.vim`. However, since I'm using a HEAD version, it can be something like `/usr/local/Cellar/neovim/HEAD-0af18a6/share/nvim/runtime/macmap.vim`. I haven't come up with anything smarter rather than just copying this file to $HOME/.config/nvim and sourcing from there.

### Environment variables

NeoVim and its plugins rely on various environment variables. You normally can configure them in your `.bashrc`or `zshrc` when you're working in your terminal app. However, it's not that obvious for GUI applications. I have found two ways to provide Env variables to NeoVim-QT. I will show you both.

#### ginit.vim

On the last 4 lines of my `ginit.vim` example I'm basicaly setting some Env vars for Perl parser. You can set any variables you want with `let` directive. Also, you can use different values in your default shell config and *rc files. So, you can override a default variable here. For now, I haven't found any disadvantages of this method.

#### /etc/launchd.conf

This file allows you to set Global Environment variables, which are accessible from all your GUI applications. Here is [an example](https://github.com/grem11n/dotfiles/blob/master/etc.launchd.conf):

```bash
setenv LANG en_US.UTF-8
setenv LC_CTYPE en_US.UTF-8
setenv TERM xterm-256color
setenv PATH /usr/local/opt/ruby/bin:/usr/local/lib/ruby/gems/3.0.0/bin:/Users/yurii.rochniak/perl5/bin:/Users/yurii.rochniak/.nvm/versions/node/v14.15.0/bin:/usr/local/Cellar/tfenv/1.0.1/bin:/Users/yurii.rochniak/.yarn/bin:/Users/yurii.rochniak/.config/yarn/global/node_modules/.bin:/Users/yurii.rochniak/.krew/bin:/Users/yurii.rochniak/Golang/bin:/Users/yurii.rochniak/Library/Python/3.8/bin:/Users/yurii.rochniak/Library/Python/2.7/bin:/usr/local/sbin:/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin:/opt/X11/bin:/usr/local/git/bin:/usr/local/MacGPG2/bin:/usr/local/go/bin:/Users/yurii.rochniak/.cargo/bin
```

This is very similar to what you would normally do in your shell configuration file with the only difference that you should use `setenv` directive on MacOS. [Here is an article, where I found it](https://www.bounga.org/tips/2020/04/07/instructs-mac-os-gui-apps-about-path-environment-variable/).

Once, you set the environment variables in `launchd.conf`, don't forget to export them in the runtime and reload your Dock and Spotlight. Alternatively, you can just reboot your laptop.

```bash
egrep "^setenv\ " /etc/launchd.conf | xargs -t -L 1 launchctl
```

```bash
killall Dock
killall Spotlight
```

You don't have to follow both approaches of setting env variables at the time. It's just good to have options.

---

Now, your GUI for NeoVim should work just fine! There might be still some issues. I recommend you checking NVim health once again in GUI this time. Once all these issues are fixed you should be good to go!

{{< figure src="/img/posts/nvim/main.png" >}}

Happy coding!
