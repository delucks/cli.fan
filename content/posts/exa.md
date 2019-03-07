---
title: "exa"
date: 2019-02-06T11:13:50-05:00
description: "exa: A modern version of `ls`"
draft: true
---

# exa

`exa` is a new iteration on `ls`. It faithfully implements common display flags to `ls` while removing some infrequently-used features and adding support for version control status & semantic terminal highlighting.

| Quick Facts | |
| ---- | ----------- |
| Version Control | git on [github](https://github.com/ogham/exa) |
| Author | Benjamin Sago ([ogham](https://github.com/ogham/)) |
| Language | Rust |

In this post, I'm using `exa` version 0.7.0 and both the stock OSX `ls` (which is probably > 10 years old) and GNU `ls`.

## Installation

`exa` is distributed as a statically compiled binary for x86_64. You can find downloads for Linux & Mac OSX on the [github release page](https://github.com/ogham/exa/releases). Other target architectures may compile using the rust toolchain.

You can also install `exa` through a few package managers. If you're using homebrew on OSX, `brew install exa`. If you're a rust developer, `cargo install exa`. If you're using the Nix package manager, `nix-env -i exa`.

## Usage

These examples are going to be performed on the checked-out [source code repo](https://github.com/delucks/cli.fan) for this blog.

For basic usage, `exa` is almost identical to `ls`. Running without arguments results in a listing of all visible files in the current directory, adding `-l` switches to long output.

```text
$ exa
archetypes
config.toml
content
data
layouts
resources
Session.vim
static
themes
TODO.md
$ exa -l
drwxr-xr-x   - jamie  6 Feb 11:14 archetypes
.rw-r--r-- 665 jamie  6 Feb 10:47 config.toml
drwxr-xr-x   - jamie  5 Feb 18:58 content
drwxr-xr-x   - jamie  5 Feb 18:56 data
drwxr-xr-x   - jamie  5 Feb 18:56 layouts
drwxr-xr-x   - jamie  5 Feb 18:58 resources
.rw-r--r--  69 jamie  6 Feb 11:14 Session.vim
drwxr-xr-x   - jamie  5 Feb 18:56 static
drwxr-xr-x   - jamie  5 Feb 18:58 themes
.rw-r--r--  60 jamie  5 Feb 19:40 TODO.md
```

`exa` also supports the same `-F` flag that `ls` does, to give an indicator of the filetype on each file. You can see the addition of slashes after each directory in the following example:

```text
$ exa
Session.vim  archetypes  config.toml  content  resources  themes
$ exa -F
Session.vim  archetypes/  config.toml  content/  resources/  themes/
```

`exa` implements many of `ls`'s most common flags:

| Flag | Meaning |
| ---- | ------- |
| `-a` | Show dotfiles (hidden files and directories) |
| `-l` | Long output with extended information about each file |
| `-1` | Show each item on its own line |
| `-r` | Reverse sort order |
| `-x` | Sort columns rather than rows in grid format |
| `-R` | Recurse into subdirectories |
| `-g` | Show group name in long output (`-l`) |
| `-i` | Show inode in long output (`-l`) |
| `-u` | Use last access time for sorting & printing |
| `-U` | Use creation time for sorting & printing |
| `-@` | Display extended attributes in long output (`-l`) |

However, some common `ls` flags have different meanings, which is confusing for newcomers. `-t` turns on sorting by time in `ls`, but in `exa` it's a selector for which timestamp field you want to show in long output. `-h` turns on human-readable sizes in `ls`, but `exa` does this by default so they repurpose the flag to add a header.

Here's some translations to make that transition easier:

| `ls` flag | `exa` equivalent |
| --------- | ---------------- |
| `-t` (sort by time) | `-s modified`    |
| `-h` (human-readable sizes) | On by default |
| `-G` (terminal color) | On by default |
| `-S` (sort by size) | `-s size` |

### New Features

Unlike `ls`, `exa` has a number of other display modes. It supports tree-style recursive listing, like the `tree` utility:

```text
$ exa -T content/
content
└── posts
   ├── exa.md
   └── introduction.md
```

The default display mode is called "grid" and can be explicitly turned on with the `-G`, `--grid` flag.

```text
$ exa -G themes/terminal/
LICENSE.md  exampleSite  layouts            package.json       source  theme.toml         yarn.lock
README.md   images       package-lock.json  postcss.config.js  static  webpack.config.js
```

The long display format also has a number of feature flags. `exa` supports printing a header, an enhancement I always wanted out of `ls`:

```text
$ exa -lh
Permissions Size User  Date Modified Name
.rw-r--r--    69 jluck  1 Mar  9:28  Session.vim
drwxr-xr-x     - jluck  6 Mar  6:58  archetypes
.rw-r--r--   779 jluck  1 Mar 10:10  config.toml
drwxr-xr-x     - jluck  1 Mar  9:28  content
drwxr-xr-x     - jluck  1 Mar  9:29  resources
drwxr-xr-x     - jluck  1 Mar  9:28  themes
```

`exa` also understands git version control status, which can be turned on with the `--git` flag:

```text
$ exa --git -l
.rw-r--r--  69 jluck  1 Mar  9:28 -- Session.vim
drwxr-xr-x   - jluck  6 Mar  6:58 -- archetypes
.rw-r--r-- 779 jluck  1 Mar 10:10 -M config.toml
drwxr-xr-x   - jluck  1 Mar  9:28 -M content
drwxr-xr-x   - jluck  1 Mar  9:29 -- resources
drwxr-xr-x   - jluck  1 Mar  9:28 -- themes
```

In the above output, config.toml & content are modified files in this git repository.

To supplement the shell's wildcard support, `exa` supports globbing ignore with the `-I` flag:

```text
$ exa -I '*o*'
archetypes  static  themes
```

Another cool `exa` feature is support for displaying timestamps in a number of different formats. The above examples show the default, but `iso`, `long-iso`, and `full-iso` are all supported as well.

```text
$ exa --time-style full-iso -l
.rw-r--r--  69 jluck 2019-03-01 09:28:30.488772561 -0800 Session.vim
drwxr-xr-x   - jluck 2019-03-06 06:58:02.198227736 -0800 archetypes
.rw-r--r-- 779 jluck 2019-03-01 10:10:40.691602867 -0800 config.toml
drwxr-xr-x   - jluck 2019-03-01 09:28:30.490107441 -0800 content
drwxr-xr-x   - jluck 2019-03-01 09:29:26.427271484 -0800 resources
drwxr-xr-x   - jluck 2019-03-01 09:28:30.490768092 -0800 themes
```

The terminal color support can also be used for weighting filesizes. I created a large file (called "largefile" because I'm creative) in the current directory to show off `--color-scale`:

![exa --color-scale](/exa_color-scale.png)

The multi-GB file which I created is much larger than the rest, so `exa` shows that using the color of the size column in long output.
