---
title: "{{ replace .Name "-" " " | title }}"
date: {{ .Date }}
description: "{{lower .Name }}: short tagline"
draft: true
---

# {{lower .Name }}

`{{lower .Name }}` is a... and you should are about it because...

![{{lower .Name }} overview](/{{lower .Name }}_main.png)

| Quick Facts | |
| ---- | ----------- |
| Version Control | vc on [provider](https://site.com/{{lower .Name }}) |
| Author | Authorname ([alias](https://author.link)) |
| Language | Lang |
| Binary Size | `Size in K, M, or G` |
| Version Reviewed | version number |

## Installation

`{{lower .Name }}` is distributed...

It's also available in the following package managers:

| Platform | Installation Command |
| -------- | -------------------- |
| Debian | `apt-get install {{lower .Name }}` |
| Ubuntu | `apt-get install {{lower .Name }}` |
| Fedora | `dnf install {{lower .Name }}` |
| CentOS/RHEL | `yum install {{lower .Name }}` |
| Alpine | `apk add {{lower .Name }}` |
| Archlinux | `pacman -S {{lower .Name }}` |
| AUR | `yaort -S {{lower .Name }}` |
| Void Linux | `xbps-install -S {{lower .Name }}` |
| Gentoo | `emerge -av {{lower .Name }}` |
| openSUSE | `zypper in {{lower .Name }}` |
| HomeBrew | `brew install {{lower .Name }}` |
| MacPorts | `port install {{lower .Name }}` |
| Nix | `nix-env -i {{lower .Name }}` |
| FreeBSD pkg | `pkg install {{lower .Name }}` |
| Scoop | `scoop install {{lower .Name }}` |
| Chocolatey | `choco install {{lower .Name }}` |
| Cargo (Source) | `cargo install {{lower .Name }}` |
| Golang | `go get {{lower .Name }}` |
| PyPI | `pip install {{lower .Name }}` |

## Usage

```text
$ {{lower .Name }}
```

## tl;dr

(re)Summarize why this tool is interesting and worth using.

:information_source: **Interested in seeing more posts like this?** Subscribe to the [cli.fan RSS feed](/posts/index.xml) linked here and in the top navigation bar.
