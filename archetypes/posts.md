---
title: "{{ replace .Name "-" " " | title }}"
date: {{ .Date }}
description: "cli.fan entry for {{ .Name }}"
draft: true
---

# {{ .Name }}

Introduction for tool {{ .Name }} goes here!

## Usage

```text
$ {{ .Name }}
```

## Links

| Link | Description |
| ---- | ----------- |
| Source Code | [github](https://github.com/foo/{{ .Name }}) |
| Author | {{ .Name }} was written by person |
| Language | {{ .Name }} is written in language |
