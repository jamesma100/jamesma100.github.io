---
layout: post
title: "Vim beyond the basics"
---

Here are some tips and tricks that have made my life easier since transforming Vim from a one-liner-change editor for my `.rc` files to an integral part of my development lifecycle.

#### Cut, copy, paste, delete
I find myself cutting and pasting a lot, and not needing to take my hand off the keyboard to highlight text is life changing.

`dd` deletes a whole line, `D` deletes everything from your cursor till the end of the line.

`yiw` (**y**ank **i**nner **w**ord) copies the current word, and similarly `ciw` (change inner word) cuts it and puts you into insert mode.
This is great for rewriting a single word.
`i`, or "inner",  means your cursor can be anywhere on the word for the command to take effect; without it, it would have to be at the beginning.

To replace a word with something you copied into your clipboard, simply enter `viwp` (**v**isual mode, **i**nner, **w**ord, **p**aste!).

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
`0` and `$` moves you to the beginning and end of a line, while `w` and `b` moves you between words.
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
To search and replace your whole file, do `:%s/<search>/<replace>/c`.
The ending `c` is used to confirm each replacement, as I find useful when the file is large and you don't want to accidentally replace something extraneous, but can be omitted.

#### Fuzzy file search
The best solution I've found is the external extension [ctrlp](https://github.com/kien/ctrlp.vim).
There is a world of settings you can fiddle with (e.g. files to exclude, maximum number of files to search, which command to use when listing files). 
