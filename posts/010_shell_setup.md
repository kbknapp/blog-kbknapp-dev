+++
title = "My Shell Setup"
slug = "shell-setup"
draft = false
template = "page.html"
date = 2021-04-02
in_search_index = true
[taxonomies]
categories = ["shell", "linux", "terminal"]
tags = ["shell", "zsh", "linux", "terminal"]
+++

Details a few steps I take to make my terminal experience a little more
pleasant. 

<!-- more -->

I spend most of my day in the terminal in one form or another. There are a few
steps I've collected over the years to make my experience more pleasant and most
importantly more efficient.

Most of the steps require some level of software to be installed or
configuration to be changed, so I can't perform these steps on all production
servers I work in. However, many of the steps are only applicable to my local
machine, and the short period of time I'm working in a remote server is minimal,
or aided by changes I made locally.

# Configuration

Before I talk about the actual steps, I should mention where I store my custom
configuration files. Manually entering these on each machine I wish to configure
would be huge pain.

Like many people, I keep all of my configuration files stored in a public
[`dotfiles`](https://github.com/kbknapp/dotfiles) repository on GitHub. It's a
bit of a mess as I only use particular parts on particular machines. So it's not
meant to be publicly read or analyzed. However, my point is that the
configuration files are stored in a central location where I can simply clone
the repository down to my machine, symlink the files or directories I care about
and want to use and go about my day.

I don't use anything like Ansible to set up my machine because there are too
many variables that control how I use a particular machine and which steps I
want to actually utilize. 

**Aside:** I actually have a [`baseline`](https://github.com/kbknapp/baseline)
set of scripts that do all the copying/symlinking for me based on a TUI dialog I
fill out. To be clear, this is in no way meant to be publicly consumed as many of
the steps are hard-coded for my specific machines.

# Notes on Distribution

I use Fedora on all my workstations, but I've also run variants of Ubuntu at
times for work, so all the steps work on either distro. The install commands
listed in this post are for Fedora. The package names may be different in Ubuntu
and I try to call those out where I'm aware of any differences.

# Improvements

These steps change some way in which I interact with the terminal, whether they
be standalone tools, or work-flows I use. All these steps are used to improve the
speed at which I work, or to better the experience. 

Most of the changes build on each-other, or other skills I've built up along the
years so they may not be widely applicable unless you're in a similar boat. For
example, using the plugin `vi-mode` in my shell combined with `fzf` searching
through my command history is lightning fast (especially when I need to edit a
previous command).

# ZSH

While the default Bash shell is fine for many cases (and ubiquitous) it
definitely has it's limitations. Many people are familiar with and prefer the
more modern Fish shell. However, I find Fish breaks too many of my existing
scripts (well, almost all of them) as well as my muscle memory, and causes me to
have to change commands when Googling around for fixes because everyone assumes
Bash (or Sh at the least).

Add to that, the thing most people ooooh and ahhhhh over with Fish, at least
initially, is the auto-suggestions and syntax highlighting, which my preferred
shell (ZSH) can do as well as you'll see soon.

ZSH is nearly fully compatible with Bash, in fact I've had to make zero changes
to any existing scripts or commands to adapt. All my Bash knowledge works in
ZSH. 

**Aside:** The one area where I've personally hit differences is when *writing*
completion scripts for programs (i.e. the menus that pop up when you hit
`<tab><tab>`). ZSH completion scripts are pretty different (and far more
functional), but I'd also wager that 99.7% of people out there are not writing
completion scripts regularly, or at all, so this should not affect them.

Since I mentioned completion scripts in the aside, programs that support ZSH
completions are *so much nicer*, than the Bash completion variants.

Here's an example of completions of `git` in Bash:

![Bash Completions](../imgs/shell_howto_bash_git.gif)

Here's an example of completions of `git` in ZSH:

![ZSH Completions](../imgs/shell_howto_zsh_git.gif)

There are a bunch of ZSH features that I love, but the few that sound the
smallest yet I probably use the most are case-insensitive paths,
command-fix-suggestions, and not using `cd` to change directories.

Those sound tiny. But they add up quickly when using the terminal constantly.

In ZSH, you can type `cat foo`, when the file is really `Foo` and ZSH will
auto-correct the path for you.

If you typo the command, and it's close enough to a real command, or
file/directory ZSH will also suggest a fix that you can accept or ignore. For
example:

```
$ gti commit -m "foo"
zsh: correct 'gti' to 'git' [nyae]? 
```

Finally, instead of typing `cd ../blah/` you can just type `../blah/`.

Even though there is so much more ZSH can do, let's just assume you're sold, how
do we install and use it? Here's my setup:

```
$ sudo dnf install zsh
  (install ZSH)

$ sudo chsh -s $(which zsh) $USER
  (change my default shell to ZSH)
```

**Note:** `chsh` is no longer available by default in some distributions. You
must have the package `util-linux-user` installed to use it.

Then I use the following `~/.zshrc` configuration:

```
ENABLE_CORRECTION="true"
COMPLETION_WAITING_DOTS="true"

export PATH=/usr/local/bin:$HOME/.local/bin:$PATH
export SSH_KEY_PATH="~/.ssh/rsa_id"
export TERM="xterm-256color"
fpath+=~/.zfunc
compinit -U
```

## Oh-My-ZSH

ZSH by itself is pretty plain, even if functional. There is a ZSH framework that
adds a ton of useful functionality called `oh-my-zsh`. It also supports things
like themeing your prompt and whatnot but I don't really use that part as I rely
on other prompt software.

To install, you clone the repository and add a few lines to your `~/.zshrc`:

```
$ git clone https://github.com/ohmyzsh/ohmyzsh ~/.oh-my-zsh
```

Then add a few lines to the *top* your ZSH configuration file:

```
export ZSH=$HOME/.oh-my-zsh
ZSH_THEME="robbyrussell"
plugins=(git systemd)
source $ZSH/oh-my-zsh.sh
```

That `ZSH_THEME` variable controls the theme, and there are a ton. However, in
the following steps I disable it in favor of other options.

The `plugins` is the next most important line. There are a ton of
[plugins](https://github.com/ohmyzsh/ohmyzsh/tree/master/plugins), and the ones
listed above just a few that apply to almost any box. In the following sections
we'll add a few more. Many plugins simply add the shell completions for
particular commands, while others add entirely new functionality. It's
definitely worth a look through the list of plugins to see if any catch your eye
for software you use frequently.

**NOTE:** There is a small startup cost associated with plugins, so it's good to
only add the ones you'll use frequently.

### Auto-Suggestions

This is the feature that most people love from Fish. It uses your command
history to suggest commands live as you type, and frankly I love it too. 

![auto-suggest](../imgs/shell_howto_auto_suggest.gif)

To add
this functionality to ZSH you clone a repo, and add a plugin.

```
$ git clone https://github.com/zsh-users/zsh-autosuggestions ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions
```

Then add `zsh-autosuggestions` to your `plugins` array in your `~/.zshrc`

### Syntax Highlighting

Another feature people love from Fish. This one colors the command you're typing
green if the command is found and valid, or red if not. This again sounds so
small, but can be such a huge help when typing commands you don't use often or
when copy/pasting from other medium. It can even catch typos *before* running
the command. I love it.

![syntax-highlight](../imgs/shell_howto_syntax.gif)

Again, a simple clone and add a plugin.

```
$ git clone git clone https://github.com/zsh-users/zsh-syntax-highlighting.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting
```

Then add `zsh-syntax-highlighting` to the `plugins` array.

### Abbrev-Alias

This plugin **expands** aliases automatically. For example, I have an alias of
`gl='git log --graph --all --oneline --decorate'`. Using this plugin, when I
type `gl` followed by `<space>` (assuming I want to add on to the command) or
`<enter>` to run the command the `gl` is replaced by the *real underlying
command*.

![abbrev-alias](../imgs/shell_howto_abbrev.gif)

Why would you want this?! Because all too often people memorize the alias, but
not the command it represents. This means when you SSH into a server, or on a
machine without all your custom aliases you can feel at a disadvantage. Using
`abbrev-alias` keeps these real commands front and center so that one at least
*sees* them as frequently as they're run.

To install, you clone a repo and `source` the script in your `~/.zshrc`, and use
`abbrev-alias` instead of the normal `alias` command.

```
$ git clone git clone https://github.com/momo-lab/zsh-abbrev-alias ~/.config/zsh-abbrev-alias/
$ echo 'source $HOME/.config/zsh-abbrev-alias/abbrev-alias.plugin.zsh' >> ~/.zshrc
```

And then for example, to add the `gl` alias I mentioned earlier

```
abbrev-alias gl='git log --graph --all --oneline --decorate'
```

Liberal use of aliases is a key aspect of quickly using the terminal. This
plugin removes the biggest detriment of why some people dislike aliases!

**NOTE:** The only time I still use normal aliases is if I'm replacing a command
with another, i.e. `alias ls=exa` **and** the flags/options on the two commands
are compatible (or nearly so).

### Sudo

Another tiny quality of life improvement is the ability to quickly add `sudo` to
the beginning of any command. If I'm half-way through typing a long command and
realize I forgot to add `sudo`, in Bash I may have to erase the command and
start over, or move the cursor to the beginning of the line to type `sudo` and
back to the end, or **if the command is safe** to run as an unprivileged user
just run it and then quickly run `sudo !!` to replay the command. However, all
of those are not great and can be slow.

This plugin allows me to simply type `<esc><esc>` which adds `sudo` and keeps my
cursor wherever I had it. As a bonus, it doesn't even negatively affect
`vi-mode` (which we'll see next!) which also uses `<esc>`.

![sudo](../imgs/shell_howto_sudo.gif)

To install simply add `sudo` to the `plugins` array as it's built in to `oh-my-zsh`.

### Vi Mode

This plugin is only useful if you're used to Vi(m) keybindings. I use it
constantly. When combined with `starship` below, this even better because you
get a visual indicator on your prompt if you're in Normal mode or Insert mode.

![vi-mode](../imgs/shell_howto_vi_mode.gif)

To install, simply add `vi-mode` to your `plugins` array as it's built in to
oh-my-zsh.

## Custom Prompts

Custom prompts allow you to visually determine facts about you're environment
without having to run additional commands. Most of the time, all these bits of
information you can find via manual commands, but to efficiently use the
terminal we try to minimize the manual commands we have to use.

Some people prefer this information in other locations like a `tmux` status
line. However, I prefer to have most of the information about my **ephemeral
environment** in my shell prompt.

'Ephemeral environment' means it can change from command to command, or
directory to directory. Non-ephemeral information is items like machine resource
information, tickers, clocks, etc. Non-ephemeral information I prefer in places
like `tmux` or my desktop environment (DE), not my prompt.

To combat the problem where long paths, or lots of bits of information can start
to take up a large portion of the current line, I prefer prompt solutions which
allow multi-line prompts where all the "status info" is on the first line, and
my cursor/command is on the second line.

There are two great solutions to prompts; Starship and Powerlevel10k (based on
Powerlevel9k). I find Powerlevel10k to be faster, but Starship is easier to
customize, has some great features, and is easier to install. I use Starship by
default, but have switched to Powerlevel10k from time to time.

### Starship

The starship prompt is run with a single binary, and configured with a single
file which is one reason I really like it.

The [demo site](https://starship.rs/) has a great GIF showing all the features
and the *why*. As well as how to configure all the various items. I have a
customized prompt, but the default is perfectly acceptable as well.

To install:

```
$ curl -fsSL https://starship.rs/install.sh | bash
```

To use, simply comment out your `ZSH_THEME` variable, and add this line to your
`~/.zshrc` (which the install script may have already added):

```
eval "$(starship init zsh)"
```

### Powerlevel10k

To install we clone a repo, set our `ZSH_THEME` and configure a `~/.p10k` file.
I find it less configurable when it comes to the actual items I want displayed,
but I also find it faster than starship at times since it was designed with ZSH
in mind (whereas starship is designed to be used with any shell).

Clone the project:

```
$ git clone git clone --depth=1 https://github.com/romkatv/powerlevel10k.git ${ZSH_CUSTOM:-$HOME/.oh-my-zsh/custom}/themes/powerlevel10k
```

Then we set our `ZSH_THEME="powerlevel10k/powerlevel10k"`.

When we open the terminal again we'll be prompted to configure a `.p10k` file
interactively.

# Tools

Now that we've set up our shell and prompt, there are a few tools I use
constantly while in the terminal. Some are related to the actual shell, while
others are just general use tools I find well worth the install.

## Terminator, Tilix, or `tmux`

To begin, I'll mention a bit a cheat. The actual terminal emulator itself. There
is one area where the terminal emulator itself can *greatly* improve one's
efficiency.

Tiling.

Both Terminator and Tilix provide "tiling" to allow one to quickly open up
another terminal to the side or below the current terminal. Terminator also
allows one to define "groups" and type in one terminal, but broadcast each
command to all terminals.

This can be a huge productivity boost over disjoint terminals that require one
to constantly move the mouse back and forth between windows, etc.

The alternative to this is if you're using a true tiling window manager such as
i3, or `tmux` to handle the tiling inside a single terminal session.

**NOTE:** Of course you *could* use `tmux` inside Tilix, inside i3...I feel like
there is an Xzibit joke there.

Many people find using Terminator or Tilix to be a good introduction to tiling
that provides a good mix between usability and functionality without fully
committing to something like `tmux` or a Tiling Window Manager.

I'd don't personally use Tilix or Terminator much, as i3 handles all the tiling
for me. I do, however, use `tmux` when SSH'ed into various servers and want to
tile panes without having to re-authenticate.

## FZF

The fuzzy finder. This tool is so amazing that it deserves a full post itself. 

It fuzzy matches through lists and allows one to interactively select one (or
multiple) items. Sounds simple.

When it comes to ZSH and the like the two features I use *constantly* are
`ctrl-r` to fuzzy search through my history and `ctrl-t` to search through file
paths.

Most package managers include the `fzf` package, and place the appropriate
configuration files. All you have to do is install it, and source the ZSH script
in your `~/.zshrc`

```
$ sudo dnf install fzf
$ echo '[ -f ~/.fzf.zsh ] && source ~/.fzf.zsh' >> ~/.zshrc
```

FZF can also be used anywhere a list is produced and you want to pick an (or
multiple) elements.

![FZF](../imgs/shell_howto_fzf.gif)

### ctrl-r

Pressing `<ctrl>-r` will bring up a list of commands which you can fuzzy match
through, press `<ctrl>-up/down` (or `<ctrl>-j/k` for vim keybindings) and hit
`<enter>` on a selection to have that command pasted into your command line.

![CTRL-R](../imgs/shell_howto_ctrl_r.gif)

It's **S-O-O-O** much better than pressing `up` two hundred times to find the
command you want in your history.

### ctrl-t

This works the same as `<ctrl>-r`, except its a list of paths. 'Nuf said.

## Navi

[`navi`](https://github.com/denisidoro/navi) is about fuzzy search-able command
cheat sheets. Yes please.

With this tool, I can search through (and create my own) command cheat sheets.
It's difficult to explain, but when I press `<ctrl>+n` a TUI pops up (also using
`fzf` by the way) which I can fuzzy match through. Upon selecting a command,
that command can have templates where I fill in the required information. Once
complete, I'm dropped back to a terminal with the filled out complete command.
Amazing!

Here's a GIF that explains it better than words can. I'm adding a new port the
firewall via `firewalld`:

![navi](../imgs/shell_howto_navi.gif)

I tend to make more generic commands, and then will tweak the invocation
manually at the end (which with `vi-mode` is super fast).

Installation is done by either building from source, or using bash:

```
$ bash <(curl -sL https://raw.githubusercontent.com/denisidoro/navi/master/scripts/install)
```

Making your own cheat sheets is beyond the scope of this post, but highly worth
it. It's also possible to use other's repositories and have them auto updated.
You can find my own sheets [in a GitHub
repository](https://github.com/kbknapp/navi-cheats)

By default, `navi` can be run from the command line normally. Setting up the
`<ctrl>-n` keybinding can be accomplished by `eval`'ing the built-in widget command in your `~/.zshrc`:

```
eval "$(navi widget zsh)"
```

By default the keybinding is `<ctrl>-g`. I prefer to just run `navi widget zsh`
and paste the output into my `~/.zshrc` and change the keybinding from `g` to
`n`, and also point it to my own cheat repositories locally.

# Conclusion

That's all for now. I could write more (and perhaps will later) about some of
the tools I use regularly in the command line, such as `exa`, Ripgrep, fd-find,
or `bat`. There's also the potential to write about the aesthetic changes I make
to improve my enjoyment of the terminal, such as font selection or which
terminal emulator I actually use.
