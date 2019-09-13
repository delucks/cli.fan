---
title: "Better find Series: fd"
date: 2019-06-18T19:48:13-04:00
description: "fd is a simple, fast and user-friendly alternative to find"
draft: true
---

# fd

This post is the first in a series evaluating tools that perform the same function as the unix utility `find`- locating files by predicate and optionally performing actions on the files returned. I chose `fd` because the argument space is relatively small and easy to remember. It's fast at what it does and gets the job done well.

![fd overview](/fd_main.png)

| Quick Facts | |
| ---- | ----------- |
| Version Control | git on [github](https://github.com/sharkdp/fd) |
| Author | David Peter ([sharkdp](https://david-peter.de/)) |
| Language | Rust |
| Binary Size | `2.2M` |
| Version Reviewed | 7.3.0 |

## Installation

`fd` is distributed as a [statically built binary](https://github.com/sharkdp/fd/releases) for x86 Windows, Linux, and Mac OSX systems. `fd` is also available in a ton of package managers:

| Platform | Installation Command |
| -------- | -------------------- |
| Debian | `apt-get install fd-find` |
| Ubuntu | `apt-get install fd-find` |
| Fedora | `dnf install fd-find` |
| Alpine | `apk add fd` |
| Archlinux | `pacman -S fd` |
| Void Linux | `xbps-install -S fd` |
| Gentoo | `emerge -av fd` |
| openSUSE | `zypper in fd` |
| HomeBrew | `brew install fd` |
| MacPorts | `port install fd` |
| Nix | `nix-env -i fd` |
| FreeBSD pkg | `pkg install fd-find` |
| Scoop | `scoop install fd` |
| Chocolatey | `choco install fd` |
| Cargo (Source) | `cargo install fd-find` |

## Usage

All these examples are going to be run within my `/usr`, which has a solid 300k files to look through. The most basic use of `fd` is with no arguments, which prints all files under the current directory recursively. Since I can't (and won't :smile:) print all 300k lines of output, I'll introduce the `--max-depth`/`-d` argument, which specifies the depth of recursion.

```text
$ fd --max-depth 1
bin
games
include
lib
libexec
local
sbin
share
src
```

This is roughly equivalent to the following invocation of `find`:

```
$ find -maxdepth 1
.
./sbin
./share
./lib
./libexec
./bin
./local
./include
./src
./games
```

A few differences are immediately apparent: `fd` does not include the path fragment `./` in relative results. `fd` also chooses the GNU longopt format `--max-depth` instead of the POSIX `-maxdepth`. Judgements of quality aside, the GNU format is more widespread these days and less intimidating to new command-line users.

The first argument to `fd` is a pattern to search among files beneath the current directory. If this path is omitted, the pattern is implicitly the empty string which matches everything.

```
$ fd -d1 bin
bin
sbin
```

This argument is a regular expression against the final path fragment by default. You can override both those defaults: to perform a non-regex search, pass the `-F`/`--fixed-strings` option, and to match against the full path use `-p`/`--full-path`.

```
$ fd -d1 's?b.n'
bin
sbin
$ fd -d1 -F 's?b.n'
```

The pattern's case sensitivity is controlled by the `-s` and `-i` options, which turn on sensitive and insensitive match respectively. The default is "smart case", which turns on case sensitivity when capitals are used in the pattern.

The second argument to `fd` is the starting directory. If this is omitted, it is implicitly `.`, the current directory. You can pass multiple values for the start directory, they will all be searched for the pattern:

```
$ fd -d1 's?b.n' / /usr
/bin
/sbin
/usr/bin
/usr/sbin
```

Both `fd` and `find` will print absolute paths if you pass an absolute path as the directory argument, reflecting that the root search directory is always the start directory. `fd` optionally allows you to print the full path with a relative search directory- the `-a`/`--absolute-path` argument enables this behavior.

Speaking of path selection, `fd` attempts to ignore paths specified in a `.gitignore` file if one is present in the start directory. If you're working outside a git repository (or just want to blacklist certain files or paths at a directory level), you can create a `.fdignore` file in the same gitignore format.

```
$ cat ~/.fdignore
*.pyc
.cache/
$ fd '\.pyc$' ~
```

You can turn off this behavior with `-I`/`--no-ignore`. Important to note: without `-H`/`--hidden`, `fd` will also not search through "hidden" files (implementation is OS-dependent).

Now that we've talked through the basic path selection features `fd` has in common with `find`, let's take a look at some of `fd`'s novel features.

## tl;dr

(re)Summarize why this tool is interesting and worth using.

:information_source: **Interested in seeing more posts like this?** Subscribe to the [cli.fan RSS feed](/posts/index.xml) linked here and in the top navigation bar.
