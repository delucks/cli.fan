---
title: "Comparison of HTTP Benchmarking Tools"
date: 2019-04-04T23:39:56-04:00
description: "Evaluating the usability and performance of three popular tools"
draft: true
---

# Web Server Load Testing

Today we're analyzing four tools that can perform a high rate of HTTP connections per second to load-test HTTP servers. We compare ApacheBench (`ab` in the rest of the post) with `siege`, `hey` and `wrk`, including veterans of the benchmarking space along with some newer tools.

There's a category of tools that are missing from this post. Projects like [locust](https://github.com/locustio/locust) and [tsung](http://tsung.erlang-projects.org/1/01/about/) are frameworks for programming load tests of web servers in python and erlang respectively. While some of the tools covered in this post have scripting extensions built in, they are much simpler and focused on load-testing a specific endpoint at a time. I wanted to have an even(ish) comparison without getting into a discussion of programming models and DSLs.

## Methodology

To test these tools against each other equally, we want to compare them in a few dimensions:

- Ease of use: is the output easy to interpret? What formats for results are supported? Are the command-line flags intuitive?
- Integration: can this tool be scripted to perform regular tests of software? Can the output be parsed by a machine?
- Performance: three different tests performed on local and cloud machines.

![http-benchmarking overview](/http-benchmarking_main.png)

| Quick Facts | `ab` | `hey` | `wrk` | `siege` |
| ---- | ----------- | --- | --- | --- |
| Version Control | svn on [apache.org](https://svn.apache.org/viewvc/httpd/httpd/trunk/support/ab.c?view=markup) | git on [github](https://github.com/rakyll/hey) | git on [github](https://github.com/wg/wrk) | git on [github](https://github.com/JoeDog/siege) |
| Author | [Adam Twiss and other authors](https://svn.apache.org/viewvc/httpd/httpd/trunk/support/ab.c?view=markup#l36) | [JBD](https://rakyll.org) | [Will Glozer](http://glozer.net) | [Jeff Fullmer](https://www.joedog.org) |
| Language | C | Go | C | C |
| Binary Size | `Size in K, M, or G` | | | |

## Installation

`ab` is distributed with the Apache web server, so install the apache package from your local package manager.

`hey` can be downloaded from the [github release page]() as a statically compiled binary.

## Ease of Use

```text
$ http-benchmarking
```

## Integration

## Performance

The following tests were performed on two VMs in a cloud hosting provider with dedicated networking hardware and a private network between them. The same tests were repeated on a pair of Odroid C2s with a gigabit link between them. We'll refer to these as "Cloud" and "Local" respectively.

- Maximum Requests per Second: create a no-op web server to test how many full requests per second could be performed by each tool. This is mostly a test of parallelism.
- Page Interaction: load testing through HTTP basic authentication and POST-ing a form. This tests performance of the tool when multiple options are passed.
- HTTP2
- Slow Server: a server designed to back up throughout the test. The variety of results should differentiate the output formats from each other.

## tl;dr

(re)Summarize why this tool is interesting and worth using.

:information_source: **Interested in seeing more posts like this?** Subscribe to the [cli.fan RSS feed](/posts/index.xml) linked here and in the top navigation bar.
