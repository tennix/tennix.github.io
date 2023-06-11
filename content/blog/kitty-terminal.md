+++
title = "Replace Tmux and Alacritty with Kitty Terminal"
date = 2022-07-31

[taxonomies]
tags = ["terminal", "tmux"]
+++

I used [Tmux](https://github.com/tmux/tmux/wiki) a lot in the past. On
macOS I run Tmux in
[Alacritty](https://github.com/alacritty/alacritty). Recently I've
replaced Alacritty + Tmux with
[Kitty](https://sw.kovidgoyal.net/kitty/). Having used Kitty for about
a month, I find it's pretty awesome and can seamlessly switch to it
with nearly the same set of keybindings as Tmux.

<!-- more -->

## Why

I once installed Kitty, but found it behaved odd with SSH and Tmux. So
I kept using Tmux in Alacritty.

Recently, I read this
[thread](https://github.com/kovidgoyal/kitty/issues/391#issuecomment-776487537)
by accident. The author of Kitty rant on Tmux and described it as
horrible hack.  I totally buy it. So I've stopped using Tmux on my
local machine and replaced Tmux + Alacritty with Kitty.

Some nice features provided by Kitty:
* Integrate with shell: view last command output in `less` by `Ctrl+Shift+g`
* Edit remote files in an existing SSH session
* GPU accelerated
* Extensibility

## How

At first, I used Kitty's builtin keybinding. Then when I saw a
colleague using Kitty + Tmux having issues copying text using tmux
keybinding, I shared the above thread link to him and introduced the
shell integration feature to him. He spent a few hours in the weekend
to configure Kitty to mimic Tmux keybinding. This really helps a lot,
since we can use Kitty on local machine but can't get rid of tmux on
remote machines. But the keybindings are totally different which is
error-prone to press keybindings.

He shared his
[configuration](https://github.com/charleszheng44/dotfiles/blob/master/kitty/kitty.conf)
with me. But he only configured a few keys, and I used more Tmux
keybindings. So I added some more Tmux keybindings to Kitty
configuration. The whole configuration is
[here](https://github.com/tennix/dotfiles/tree/main/kitty). The keybindings are copied from [Tmux
keybinding cheatsheet](https://tmuxcheatsheet.com/).

The result is pretty exciting, it covers all my keybinding usage in
Tmux. Now I use the same set of tmux keybindings on local machine and
remote VMs, locally use kitty, remotely use tmux with `ctrl+b` as
keybinding prefix.

## Bonous

Kitty can be extended with
[kitten](https://sw.kovidgoyal.net/kitty/kittens_intro/#kittens) using
Python. Below are some useful kittens.

### [SSH Kitten](https://sw.kovidgoyal.net/kitty/kittens/ssh/)

Set an alias for it: `alias s="kitty +kitten ssh"`, then `s
my-vps-server` can connect to `my-vps-server`.

Kitty supports reusing the existing SSH session. When in the SSH
session window, invoke `new_window_with_cwd` would open a new SSH
window with the existing SSH connection and cd into the same remote
directory. This is very similar to running tmux on remote machine and
then use tmux to open new pane.

I bind `new_window_with_cwd` to `ctrl+a f` (`f` is taken from fork
which is easier to remember and not used by tmux keybinding).

After login to a remote machine using SSH kitten, I can also edit
remote files using local `$EDITOR` by:
1. `ls --hyperlink=auto`
2. Hold `Ctrl+Shift`, then click the name of the file.
3. Press `E` to edit the file using local `$EDITOR`

### [Hints Kitten](https://sw.kovidgoyal.net/kitty/kittens/hints/)

This is a very powerful kitten.

* `ctrl+shift+p n`: open path or filename followed by a colon
* `ctrl+shift+p f`: select a path or filename and then paste in terminal
* `ctrl+shift+p y`: open hyperlinks


### [Hyperlinked Grep Kitten](https://sw.kovidgoyal.net/kitty/kittens/hyperlinked_grep/)

After adding the following config to `$HOME/.config/kitty/open-actions.conf`

```config
# Open any file with a fragment in vim, fragments are generated
# by the hyperlink_grep kitten and nothing else so far.
protocol file
fragment_matches [0-9]+
action launch --type=overlay vim +${FRAGMENT} ${FILE_PATH}

# Open text files without fragments in the editor
protocol file
mime text/*
action launch --type=overlay ${EDITOR} ${FILE_PATH}
```

Then run `kitty +kitten hyperlinked_grep something` to search
`something` and then hold `ctrl+shift` to click the match to open it
in vim.

Add an alias for it `alias hg="kitty +kitten hyperlinked_grep"`
