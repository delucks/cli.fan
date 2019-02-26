---
title: "exa"
date: 2019-02-06T11:13:50-05:00
description: "exa: A modern version of `ls`"
draft: true
---

# exa

`exa` is a new iteration on the `ls` idea. `exa` lists directories, just like `ls`, but adds a slew of other features. `exa` performs semantic highlighting of directory results based on the type of the file, exceeding the capabilities of `LS_COLORS`.

| Information | |
| ---- | ----------- |
| Source Code | [github](https://github.com/ogham/exa) |
| Author | Benjamin Sago (ogham) |
| Language | Rust |

## Usage

These examples are going to be performed on the checked-out [source code repo](https://github.com/delucks/cli.fan) for this blog.

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
