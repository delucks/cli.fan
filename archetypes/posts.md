---
title: "{{ replace .Name "-" " " | title }}"
date: {{ .Date }}
description: "{{ .Name }}: short tagline"
draft: true
---

# {{ .Name }}

`{{ .Name }}` is a... and you should are about it because...

![{{ .Name }} overview](/{{ .Name }}_main.png)

| Quick Facts | |
| ---- | ----------- |
| Version Control | vc on [provider](https://site.com/) |
| Author | Authorname ([alias](https://author.link)) |
| Language | Lang |
| Binary Size | `Size in K, M, or G` |

## Installation

`{{ .Name }}` is distributed...

## Usage

```text
$ {{ .Name }}
```

## tl;dr

(re)Summarize why this tool is interesting and worth using.
