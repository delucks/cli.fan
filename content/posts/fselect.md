---
title: "fselect"
date: 2019-09-13T17:35:40-04:00
description: "fselect: Find files with SQL-like queries"
draft: false
---

# Find files with SQL-like queries

`fselect` is an alternative to `find` that presents a SQL-like interface to select files recursively. Compared to other "filesystem as SQL" projects (notably, [osquery](https://github.com/osquery/osquery)), `fselect` has a tighter scope: returning files and metadata from the filesystem. It doesn't seek to solve system management problems and instead focuses on being able to return data quickly in multiple formats. In my opinion this makes `fselect` a better tool for your everyday Unix toolbox.

This post relies on a basic understanding of SQL, specifically SQL "SELECT" queries. If you've never encountered SQL before, this [Khan Academy](https://www.khanacademy.org/computing/computer-programming/sql) video seems like a good starting point.

![fselect overview](/fselect_main.png)

| Quick Facts | |
| ---- | ----------- |
| Version Control | git on [github](https://github.com/jhspetersson/fselect) |
| Author | [jhspetersson](https://github.com/jhspetersson/) |
| Language | Rust |
| Binary Size | `5.0M` |
| Version Reviewed | 0.6.5 |

## Installation

`fselect` is distributed as a statically compiled executable for Linux, Mac, and Windows. It's also available in the following package managers:

| Platform | Installation Command |
| -------- | -------------------- |
| AUR | `yaort -S fselect` |
| HomeBrew | `brew install fselect` |
| Chocolatey | `choco install fselect` |
| Cargo (Source) | `cargo install fselect` |

## Usage

As mentioned earlier, `fselect` gives you a SQL-like interface to the filesystem. You can think of the `fselect` binary as the `SELECT` part of a SQL statement, so the second argument to `fselect` is the data you would like to "select" from each file underneath the current directory tree.

Case sensitivity doesn't matter in `fselect`'s SQL-like functionality.

For an equivalent invocation to `find .`, try `fselect path`, which selects the full path of every file underneath the current directory.

```text
$ diff <(find .) <(fselect path)
1d0
< .
```

`fselect` defaults to the current directory as the starting point for search but you can pass another directory using `from /path/to/dir`, as if the directory was a SQL table.

```
$ fselect path from /media/
/media/cdrom
```

### Fields and Columns

The "column selection" section supports multiple fields separated by commas. For example, you can get the filesize in bytes with the `size` column.

```
$ fselect path, size from /usr limit 10
/usr/sbin       12288
/usr/sbin/cupsaddsmb    14328
/usr/sbin/pppoe-discovery       22528
/usr/sbin/exim_lock     18424
/usr/sbin/update-passwd 31136
/usr/sbin/genccode      10608
/usr/sbin/netscsid      118688
/usr/sbin/split-logfile 2415
/usr/sbin/cache_dump    11
/usr/sbin/cpgr  4
```

This also introduces the `limit` clause, which maps back directly to the SQL equivalent. I'll be using the `limit` clause extensively for the rest of this post.

In `fselect`, columns roughly map to pieces of metadata about each file under the root directory. This set of columns is gleaned using Rust's [`fs::metadata`](https://github.com/jhspetersson/fselect/blob/0.6.5/src/util/mod.rs#L284) function, which corresponds to a `stat` call on *nix systems and a `GetFileAttributesEx` call on Windows. If you request other columns that correspond to more rich metadata in the files themselves, `fselect` will open those files for additional information. For example, if you use the `is_shebang` column, `fselect` opens each file and [checks the first two bytes of the file for `#!`](https://github.com/jhspetersson/fselect/blob/0.6.5/src/util/mod.rs#L415):

```
$ fselect path from /usr where is_shebang=1 limit 10 | xargs -L 1 head -1 | sort | uniq -c | sort -rn
      5 #!/usr/bin/perl -w
      2 #!/usr/bin/perl
      1 #!/bin/sh -e
      1 #!/bin/sh
      1 #!/bin/bash
```

`fselect` will open the metadata blocks on images, music, and video to return id3 tags, EXIF data, etc when you use the relevant columns. In this post I'll be focusing on columns that aren't derived from media files.


Here's a taste of some of the string columns that `fselect` provides. Many boolean columns are available as well, like `is_shebang`. A full list of the available columns is [in the source code here](https://github.com/jhspetersson/fselect/blob/0.6.5/src/field.rs#L89).

`fselect name, path, abspath, size, fsize, uid, gid, accessed, created, modified, mode, user, group, sha1, sha256, sha512 from /usr limit 1 into json | jq .`

```json
[
  {
    "path": "./sbin/cupsaddsmb",
    "name": "cupsaddsmb",
    "created": "",
    "user": "root",
    "group": "root",
    "size": "14328",
    "mode": "-wxr--r-x",
    "gid": "0",
    "accessed": "2019-06-29 17:24:51",
    "formattedsize": "13.99 KiB",
    "sha512": "b2337c2818ddd95614704a190f522352f893e89507674f88305766f271ac118f42cc1b8f3ccddec353e8b4e8e231ed3561361c0c7e867786a178a9bcf80862ee",
    "sha1": "ba5bf8ceb7159fec5798bab0b6584539c2c9e595",
    "uid": "0",
    "sha256": "6cb0e7fb83a14d9c830909c5f3664ce6596ca1e1cd8eff0465f73174984b6e8f",
    "abspath": "/usr/sbin/cupsaddsmb",
    "modified": "2018-07-12 12:48:48"
  }
]
```

When selecting a column, you can sort results by that column with SQL-like "order by" syntax. For example, here's an invocation of `fselect` that finds the ten largest files in my `/usr`:

```
$ fselect path, size from /usr order by size desc limit 10
/usr/lib/jvm/java-9-openjdk-amd64/lib/modules   157113207
/usr/lib/chromium/chromium      115077616
/usr/lib/thunderbird/libxul.so  73204744
/usr/lib/libreoffice/program/libmergedlo.so     66122264
/usr/lib/x86_64-linux-gnu/libLLVM-6.0.so.1      62415088
/usr/lib/x86_64-linux-gnu/libQt5WebKit.so.5.212.0       47348176
/usr/lib/x86_64-linux-gnu/libwebkit2gtk-4.0.so.37.33.6  44597336
/usr/lib/python3.6/config-3.6m-x86_64-linux-gnu/libpython3.6m.a 37997006
/usr/lib/python3.6/config-3.6m-x86_64-linux-gnu/libpython3.6m-pic.a     37145502
/usr/local/bin/mpv      28791120
```

The `order by` clause takes a column name to sort by and optionally "desc" to change the sort order from ascending to descending. (You can pass "asc" to keep the sort order ascending, too) Make sure that you add asc/desc after each column, it won't apply to the whole ordering clause globally! To illustrate the sorting behavior, I created a directory with similarly named but differently sized files, with one pair of identical size.

```
$ mkdir /tmp/fselect_test; cd /tmp/fselect_test
$ for filesize in 100 1000 10000 100000; do
> dd if=/dev/urandom of="testfile_$filesize" bs=1024 count=$filesize
> done
$ dd if=/dev/urandom of=testfile_11111 bs=1024 count=10000
$ exa -l
.rw-r--r-- 102k delucks 29 Jun 13:35 testfile_100
.rw-r--r--  10M delucks 29 Jun 13:35 testfile_10000
.rw-r--r--  10M delucks 29 Jun 13:45 testfile_11111
.rw-r--r-- 102M delucks 29 Jun 13:35 testfile_100000
```

Here I'm using [`exa`](https://cli.fan/posts/exa/) to show off the relative filesizes. Let's use them to show how `fselect` handles sorting:

```
$ fselect name, size from /tmp/fselect_test/ order by size desc, name
testfile_100000       102400000
testfile_10000        10240000
testfile_11111        10240000
testfile_100  102400
$ fselect name, size from /tmp/fselect_test/ order by size desc, name desc
testfile_100000       102400000
testfile_11111        10240000
testfile_10000        10240000
testfile_100  102400
```

The order of names flipped between these two examples despite the `size` sort remaining stable.

Occasionally you will encounter columns that can't be sorted by name, for example those returned by `fselect`'s built-in aggregation functions. You can refer to columns by index, starting at 1- so these two invocations would be equivalent:

```
fselect name, size from /tmp/fselect_test/ order by name, size
fselect name, size from /tmp/fselect_test/ order by 1, 2
```

### Other SQL-Like Features

`fselect` implements a couple other features that are familiar to a user of SQL. One that's already been used in this post is the `WHERE` clause, which allows you to filter results based upon a boolean expression. An example would be limiting results by filesize:

```
$ fselect 'path, size from /usr where size > 100M'
/usr/lib/jvm/java-9-openjdk-amd64/lib/modules   157113207
/usr/lib/chromium/chromium      115077616
```

This `>` operator can also be referred to with "gt", which makes it easier to type out without quoting in the shell.

The where clause is also useful for doing regular expression matches against string columns:

```
$ fselect path where name =~ '.*note.*\.txt$'
notes/mentorship_program_notes_20150415.txt
notes/t5220_notes.txt
notes/keyboard_notes.txt
```

`fselect` also includes a bunch of built-in functions that are similar to SQL functions like SUM, LENGTH, and COUNT. Want to find the longest 10 filenames in `/usr`?

```
$ fselect 'length(name), path from /usr order by 1 desc limit 10'
108	/usr/share/qt5/doc/qtwebengine/qtwebengine-3rdparty-fiat-crypto-synthesizing-correct-by-construction-code-for-cryptographic-primitives.html
87	/usr/share/qt5/doc/qtwebengine/qtwebengine-3rdparty-breakpad-an-open-source-multi-platform-crash-reporting-system.html
86	/usr/share/qt5/doc/qtwebengine/qtwebengine-webengine-recipebrowser-resources-pages-assets-3rdparty-marked-min-js.html
86	/usr/share/qt5/doc/qtwebengine/qtwebengine-3rdparty-google-fork-of-khronos-reference-front-end-for-glsl-and-essl.html
85	/usr/share/qt5/doc/qtwebengine/qtwebengine-webengine-recipebrowser-resources-pages-assets-3rdparty-markdown-css.html
85	/usr/share/icons/Papirus/32x32/mimetypes/gnome-mime-application-vnd.openxmlformats-officedocument.presentationml.slideshow.svg
85	/usr/share/icons/Papirus/48x48/mimetypes/gnome-mime-application-vnd.openxmlformats-officedocument.presentationml.slideshow.svg
85	/usr/share/icons/Papirus/22x22/mimetypes/gnome-mime-application-vnd.openxmlformats-officedocument.presentationml.slideshow.svg
85	/usr/share/icons/Papirus/64x64/mimetypes/gnome-mime-application-vnd.openxmlformats-officedocument.presentationml.slideshow.svg
85	/usr/share/icons/Papirus/24x24/mimetypes/gnome-mime-application-vnd.openxmlformats-officedocument.presentationml.slideshow.svg
```

Those are some pretty long filenames! For a less contrived example, let's find the average size of the video files in my home directory:

```
$ fselect "SUM(size), COUNT(*) from /home/delucks where is_video=1" | tr '\t' '/' | bc
13361869
```

If I had read the list of all functions before coming up with that example, I'd find there's a built in mean function:

```
$ fselect "AVG(size) from /home/delucks where is_video=1"
13361869
```

Note that results of these functions cannot be the inputs to other functions:

```
$ fselect "YEAR(modified) from Downloads" | sort | uniq -c | sort -rn
    947 2018
    228 2016
    171 2015
     68 2010
     63 2014
     47 2013
     33 2019
$ fselect "MIN(YEAR(modified)) from Downloads"
-1
$ fselect "MAX(YEAR(modified)) from Downloads"
0
```

Here's a list of all the [built-in numeric and aggregation functions](https://github.com/jhspetersson/fselect/blob/0.6.5/src/function.rs#177) in version 0.6.5.

### Output Formats

All the output we've seen so far has been using the default output format, tab-separated values. `fselect` comes with a few other output modes, which we can illustrate via a simple query that returns little data:

```
$ fselect '*' from bin where is_binary=1 limit 2
-wxr--r-x       delucks   delucks   3173528 bin/broot
rwxrwxrwx       delucks   delucks   35      bin/multitool
```

This is the default "tabs" output format we've already seen. Let's try some other line-oriented ones:

```
$ fselect '*' from bin where is_binary=1 limit 2 into csv
-wxr--r-x,delucks,delucks,3173528,bin/broot
rwxrwxrwx,delucks,delucks,35,bin/multitool
$ fselect '*' from bin where is_binary=1 limit 2 into lines
-wxr--r-x
delucks
delucks
3173528
bin/broot
rwxrwxrwx
delucks
delucks
35
bin/multitool
```

That's not very webscale, now is it?

```
$ fselect '*' from bin where is_binary=1 limit 2 into json | jq .
[
  {
    "size": "3173528",
    "mode": "-wxr--r-x",
    "user": "delucks",
    "group": "delucks",
    "path": "bin/broot"
  },
  {
    "user": "delucks",
    "path": "bin/multitool",
    "group": "delucks",
    "mode": "rwxrwxrwx",
    "size": "35"
  }
]
$ fselect '*' from bin where is_binary=1 limit 2 into html
<html><body><table><tr><td>-wxr--r-x</td><td>delucks</td><td>delucks</td><td>3173528</td><td>bin/broot</td></tr><tr><td>rwxrwxrwx</td><td>delucks</td><td>delucks</td><td>35</td><td>bin/multitool</td></tr></table></body></html>
```

Here's the table `fselect` created:

<html><body><table><tr><td>-wxr--r-x</td><td>delucks</td><td>delucks</td><td>3173528</td><td>bin/broot</td></tr><tr><td>rwxrwxrwx</td><td>delucks</td><td>delucks</td><td>35</td><td>bin/multitool</td></tr></table></body></html>

## Benchmarking

First let's look at the two commands simplest forms, ran over a 6.3G /usr directory with about 400k files, with warm caches:

```
$ hyperfine "find ." "fselect path"
Benchmark #1: find .
  Time (mean ± σ):     341.9 ms ±   5.3 ms    [User: 103.0 ms, System: 235.9 ms]
  Range (min … max):   335.2 ms … 354.8 ms    10 runs

Benchmark #2: fselect path
  Time (mean ± σ):      1.666 s ±  0.071 s    [User: 955.3 ms, System: 707.9 ms]
  Range (min … max):    1.590 s …  1.811 s    10 runs

Summary
  'find .' ran
    4.87 ± 0.22 times faster than 'fselect path'
```

`find` is the clear victor here, both in terms of overall execution time and CPU time. Let's try again with filesystem caches dropped:

```
$ hyperfine --prepare 'sync; echo 3 | sudo tee /proc/sys/vm/drop_caches' "find ." "fselect path"
Benchmark #1: find .
  Time (mean ± σ):      3.599 s ±  0.143 s    [User: 182.8 ms, System: 765.9 ms]
  Range (min … max):    3.421 s …  3.858 s    10 runs

Benchmark #2: fselect path
  Time (mean ± σ):      6.233 s ±  0.136 s    [User: 1.329 s, System: 1.890 s]
  Range (min … max):    6.079 s …  6.541 s    10 runs

Summary
  'find .' ran
    1.73 ± 0.08 times faster than 'fselect path'
```

`find` still comes out on top by a wide margin, but through similar tests to this one I've found that the performane of the two tools is more comparable with caches off. It's possible `find` has optimizations around how it loads the filesystem cache, or that `fselect` is loading enough data that the caches are frequently overwritten.

Some of `fselect`'s relative slowness is due to the decades of amazing optimizations that have made their way into `find`. One example: `find` will stop calling `stat` on entries of a directory when it contains 2 fewer subdirectories than its hard link count. This "leaf" optimization, turned off with `-noleaf`, means that `find .` executes drastically faster by avoiding calls to `stat` on a large count of directory entries.

Let's see the relative speed of each tool when they're both coerced into calling `stat` on every file. To do this, I'm disabling the optimization noted above and instructing `find` to print the file's last access time, a field returned by `stat`. First, with warm filesystem caches:

```
$ hyperfine -w 1 "fselect access" "find . -noleaf -printf '%a\n'"
Benchmark #1: fselect access
  Time (mean ± σ):     925.4 ms ±   8.2 ms    [User: 385.2 ms, System: 537.4 ms]
  Range (min … max):   916.5 ms … 945.3 ms    10 runs

Benchmark #2: find . -noleaf -printf '%a\n'
  Time (mean ± σ):     961.1 ms ±   4.5 ms    [User: 301.2 ms, System: 657.0 ms]
  Range (min … max):   952.1 ms … 967.3 ms    10 runs

Summary
  'fselect access' ran
    1.04 ± 0.01 times faster than 'find . -noleaf -printf '%a\n''
```

Now again, with the caches dropped before each program execution:

```
$ hyperfine --prepare 'sync; echo 3 | sudo tee /proc/sys/vm/drop_caches' "fselect access" "find . -noleaf -printf '%a\n'"
Benchmark #1: fselect access
  Time (mean ± σ):      7.203 s ±  0.282 s    [User: 981.9 ms, System: 2561.9 ms]
  Range (min … max):    6.566 s …  7.510 s    10 runs

Benchmark #2: find . -noleaf -printf '%a\n'
  Time (mean ± σ):      7.289 s ±  0.153 s    [User: 853.4 ms, System: 2791.1 ms]
  Range (min … max):    6.949 s …  7.433 s    10 runs

Summary
  'fselect access' ran
    1.01 ± 0.05 times faster than 'find . -noleaf -printf '%a\n''
```

Results are consistent in both cases, `fselect` is every so slightly faster than `find` when they're both forced to stat every file.

Generally, `find` is faster than `fselect`. From the tests presented above and others I didn't share in this post, I think this is due to better use of filesystem caches and the steady pace of optimization in `find`.

I'd recommend using `find` if you have a performance-critical use-case. If you're writing an application that needs tight performance when searching for files, these tools may not be the best fit since they're written more generically; I'd recommend writing code in a low-level language that optimizes for the pecularities of the files you're trying to locate. In terms of interactive use, `fselect` is fast enough that you shouldn't notice the difference.

## tl;dr

`fselect` presents an alternative approach to locating files recursively and exposes many different pieces of metadata for narrowing down your search. The SQL-like syntax has wider appeal than `find`'s CLI syntax, which is notoriously confusing to users that don't come from a unix background. The performance of `fselect` is comparable to an unoptimized `find`, making it suitable for interactive use without annoyance. If you're considering a new tool for digging through your collection of files, `fselect` is a worthy addition to your bin folder.

:information_source: **Interested in seeing more posts like this?** Subscribe to the [cli.fan RSS feed](/posts/index.xml) linked here and in the top navigation bar.
