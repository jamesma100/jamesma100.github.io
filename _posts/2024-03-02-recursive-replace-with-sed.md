---
layout: post
title: "Recursive replace with sed"
---

It would always bother me that the venerable [sed](https://en.wikipedia.org/wiki/Sed) can't work recursively on a directory.
Something common I like to do is "replace all occurrences of _x_ with _y_ in all files under some directory _d_."

So the workaround is usually 1. get the files I want and then 2. pipe those back to `sed`, which is a bit tedious.
Ideally I should be able to do something like `sed -i 's/this/that/g' my_dir`.

Here is an attempt at that - you can find the entire script [here](https://github.com/jamesma100/rsed/tree/main). First, we parse the arguments, storing the directory, the last arg, in `dir` and the rest in `sedargs`:
```
args=("$@")
dir="${args: -1}"
sedargs="${args[@]:0:${#args[@]}-1}"
```

Then, we save the sed command into `sedstr` while handling the `-i` option for "in place replacement".
```
first="${args[0]}"
if [[ $first = "-i" ]]; then
  sedstr="${args[@]:1:${#args[@]}-2}";
else
  sedstr="${args[@]:0:${#args[@]}-1}";
fi
```
(Note that the `-i` option is actually a GNU extension and is not POSIX standard.)

Next, we get the string to be searched and replaced. Assume the backslash ("/") delimiter:
```
IFS='/' read -r -a search_arr <<< "$sedstr"
to_search="${search_arr[1]}"
```

Lastly, we retrieve all files containing `to_search` and pipe them to the final `sed` invocation.
Here I am using [ripgrep](https://github.com/BurntSushi/ripgrep), which is essentially acting as a single replacement for the classic `find`/`grep` duo.
```
rg_res=$(rg -l $to_search)
sed ${sedargs[@]} $rg_res
```

And now to test, save everything into a script (I'm calling it `rsed` for "recursive sed"), and voila:
```
$ tree temp
temp
├── dir1
│   └── file2
└── file1

2 directories, 2 files
$ echo "hello, son" > temp/file1
$ echo "how are you, son" > temp/dir1/file2
$ rsed -i 's/son/child/g' temp
$ cat temp/file1
hello, child
$ cat temp/dir1/file2
how are you, child
```
The non-script alternative is to do something like this:
```
$ rg -l "child" ./temp | xargs sed -i 's/child/man/g'
```
which also works, but it is always nice to type fewer characters.

### Closing thoughts
Though this may seem like a lot of work for a trivial task, the omission of excessive features unrelated to the core task of stream-editing a file might actually be a good thing, as it adheres to the Unix philosophy of "do one thing well."
I'd be pretty annoyed if I was maintaining some project and users kept asking for bells and whistles that can easily be achieved elsewhere.

To that point, here is a complaint from Brian Kernighan from _Unix: A History and a Memoir_ that captures this sentiment pretty well:

> These are maxims to program by, though not always observed. One example: the `cat` command that I mentioned in Chapter 3. That command did one thing, copy input files or the standard input to the standard output. Today the GNU version of `cat` has (and I am not making this up) 12 options, for tasks like numbering lines, displaying non-printing characters, and removing duplicate blank lines. All of those are easily handled with existing programs; none has anything to do with the core task of copying bytes, and it seems counter-productive to complicate a fundamental tool.
