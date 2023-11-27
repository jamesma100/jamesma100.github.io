---
layout: post
title: "Vim beyond the basics"
---

Here are some tips and tricks that have made my life easier since transforming Vim from a one-liner-change editor for my `.rc` files to an integral part of my development lifecycle.

#### Cut, copy, paste, delete
I find myself cutting and pasting a lot, and not needing to take my hand off the keyboard to highlight text is life changing.

For text deletion, `dd` deletes a whole line, `D` deletes everything from your cursor till the end of the line.

For copying/replacing, `yiw` (**y**ank **i**nner **w**ord) copies the current word, and similarly `ciw` (change inner word) cuts it and puts you into insert mode.
This is great for rewriting a single word.
`i`, or "inner",  means your cursor can be anywhere on the word for the command to take effect; without it, it would have to be at the beginning.

To replace a word with something you copied into your clipboard, simply enter `viwp` (**v**isual mode, **i**nner, **w**ord, **p**aste!).
A common pattern is to copy a single word and then replace some other word with what you just copied.
In that case, you just need a `yiw` on the first word, followed by a `viwp` on the second.

All these commands are able to be simple and deducible because Vim has great composibility.

#### Registers
Whenever you copy or cut something, it goes into your default register, and can be fetched with `p` (paste after current line)  or `P` (paste before current line).
This means if you copy something with `y` then cut something with `c`, pasting with `p` will paste the latter, seemingly overwriting the former.
There is, however, a dedicated yank register, and you can get its contents with `"0p`.

For example, if we yank `yanktext` then cut `cuttext`, `:reg` shows:

```
:reg
Type Name Content
  c  ""   cuttext
  c  "0   yanktext
...
```
Regular `p` will grab what's in the default register, `cuttext`, while the yanked text can be accessed with `"0p`. There are many more registers available, as you can see with `:reg`, and you can get the contents of a specific register with `"<reg num>p`.


#### Cursor movement
Some shortcuts I like include `0` and `$`, which move you to the beginning and end of a line, as well as `w` and `b`, which move you between words.
Hit `gg` and `G` to move to the beginning and end of the file. `ctrl-o` gets you to your previous position.
This also works between Vim sessions, meaning if you quit Vim after editing a file and reopen it, your previous position within the file is still saved.
Another feature I like when writing deeply nested code is `%`, which jumps to the matching bracket or parenthesis.

For even better cursor line movement, I enable `set number relativenumber` in my `.vimrc` to make line numbers relative:
```
  2 Remember!
  1 Blue cat jumped
43  over red dog.
  1 That's crazy!
```
So jumping up 2 lines is a matter of typing `2k`.
A huge win over potentially typing `k` ten times, or even worse, moving your mouse up ten lines!

#### Tabs and windows
You can use tabs and windows within a single Vim session. `:tabnew <optional filename>` opens a new tab, and `gt` and `gT` switches you to your previous and next tabs, respectively, while `:tabclose` closes your tab.

A horizontal window can be created with `:split <filename>`, and a vertical window is similarly `:vsplit <filename>`.
You can toggle between multiple windows with `ctrl-w <up/down/left/right`, or cycle between them (my preference!) with `ctrl-w ctrl-w`.
Finally, `:hide` closes the current window.

#### Search and replace
Searching is simply `/<something>` then `n`/`N` to move forward/backward.
You can also type `*` to find the word your cursor is currently on.
To search and replace your whole file, do `:%s/<search>/<replace>/c`.
The ending `c` is used to confirm each replacement, as I find useful when the file is large and you don't want to accidentally replace something extraneous, but can be omitted.

#### Fuzzy file search
If you work on a large project, you might need fuzzy search.
The best solution I've found is the external extension [ctrlp](https://github.com/kien/ctrlp.vim).
There is a world of settings you can fiddle with (e.g. files to exclude, maximum number of files to search, which command to use when listing files). 

#### Command line integration
You can run shell commands with `:!`, e.g. `:!echo "hello world"` but I don't really do this, as you can't use the keybindings (or any of Vim's features).
My preferred way of running commands is via `q:` (or `:ctrl-f`) - there you can see your recent shell history.
A basic history looks something like:
```
:  3 :h cmdline-window
:  2 echo "hello world"
:  1 q!
:51
```
The history displayed is just regular text that you can navigate, cut/copy, edit, delete, etc. - all from within Vim!
To run `echo "goodbye world"`, for example,  you can just navigate up 2 lines, edit the existing "hello world" command, then hit enter. 
Finally, you can `:q` or `ctrl-c` to quit the integrated shell.

If you need a separate window to run your terminal, you can also use Vim's keybindings by setting `set -o vi` in your `~/.bashrc`.
Though note that this only supports navigation, so you can't, for instance, copy and paste with `y` and `p`.
