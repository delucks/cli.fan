---
title: "progress"
date: 2019-10-01T09:32:35-04:00
description: "progress: Coreutils Progress Viewer"
draft: true
---

# progress

`progress` is a small Linux application which reports the progress of file write operations initiated by coreutils programs like `cp`, `mv`, `dd`, and several others.

![progress overview](/progress_main.png)

| Quick Facts | |
| ---- | ----------- |
| Version Control | git on [github](https://github.com/Xfennec/progress) |
| Author | ([Xfennec](https://github.com/Xfennec)) |
| Language | C |
| Binary Size | `56K` |
| Version Reviewed | 0.14 |

## Installation

`progress` is installed via source distribution and `make`, like so:

```
git clone https://github.com/Xfennec/progress
cd progress/
make
make install
```

There's one dependency, on the library ncurses- you may already have the development headers installed, but if you don't it may be called `ncurses-devel` or `libncurses-dev`.

Unfortunately since `progress` relies on the `/proc` pseudo-filesystem, this utility only works on Linux.

## Usage

Running `progress` with no arguments prints out any currently-running coreutils processes that are transferring data:

```text
$ progress
```

Whoops, I don't have any running. Let's set up a script that is constantly reading and writing files, so we can illustrate `progress` on some relatively common operations. The following bash script will create a few files between 1 and 10 gigabytes, copy them to `/tmp`, then remove the files in `/tmp` and remove the original files.

INCLUDE test_progress.sh here

Now that we have some operations to report, let's try `progress`:

```
$ progress
[20293] dd /home/jamie/dev/cli.fan/File_1000000
        100.0% (440.1 MiB / 440.1 MiB)

[20297] dd /home/jamie/dev/cli.fan/File_2000000
        100.0% (433.2 MiB / 433.2 MiB)

[20301] dd /home/jamie/dev/cli.fan/File_10000000
        100.0% (413.7 MiB / 413.7 MiB)

```

There we go, three parallel `dd` operations as the files were being created. Upon seeing this I immediately noticed they're each showing 100% progress on the file write, which seems inaccurate given their differing filesizes. That's because the origin file for each of these `dd` operations was a character special file, `/dev/urandom`, and not an actual file. The special file does not have a constant length, so `progress` can only report the amount of data that's already been transferred. You'll find some other situations don't have full filesizes reported by `progress`; the most common one I've encountered is downloading a file from the network.

You can see what this _should_ look like a bit below, when we copy these files to a new location.

`progress` by default just reports the percentage and size of each file being copied, but it can also estimate a rate of data transfer with the `-w`/`--wait` flag. This will wait a set period of time to measure how much data was transferred in that interval. You can change the interval with the `-W`/`--wait-delay` argument, whose single parameter is the count of seconds in the interval. Here I'm trying the [default wait time](https://github.com/Xfennec/progress/blob/master/progress.c#L84) of 1, then increasing that to 5 seconds.

```
$ progress -w
[20293] dd /home/jamie/dev/cli.fan/File_1000000
        100.0% (829.1 MiB / 829.1 MiB) 6.9 MiB/s

[20297] dd /home/jamie/dev/cli.fan/File_2000000
        100.0% (823.4 MiB / 823.4 MiB) 6.9 MiB/s

[20301] dd /home/jamie/dev/cli.fan/File_10000000
        100.0% (816.7 MiB / 816.7 MiB) 5.9 MiB/s

$ progress -w -W 5
[20293] dd /home/jamie/dev/cli.fan/File_1000000
        100.0% (889.8 MiB / 889.8 MiB) 6.8 MiB/s

[20297] dd /home/jamie/dev/cli.fan/File_2000000
        100.0% (884.4 MiB / 884.4 MiB) 7.0 MiB/s

[20301] dd /home/jamie/dev/cli.fan/File_10000000
        100.0% (877.8 MiB / 877.8 MiB) 6.8 MiB/s

```

You can see all the detail that was missing from the `dd` commands in the following `cp` commands:

```
$ progress -w
[22791] cp /home/jamie/dev/cli.fan/File_2000000
        86.0% (1.6 GiB / 1.9 GiB) 18.1 MiB/s remaining 0:00:15

[22792] cp /home/jamie/dev/cli.fan/File_10000000
        17.1% (1.6 GiB / 9.5 GiB) 20.5 MiB/s remaining 0:06:34

$ progress -w -W 5
[22791] cp /home/jamie/dev/cli.fan/File_2000000
        98.3% (1.9 GiB / 1.9 GiB)

[22792] cp /home/jamie/dev/cli.fan/File_10000000
        25.7% (2.4 GiB / 9.5 GiB) 120.1 MiB/s remaining 0:01:00

```

Not only does `progress` calulculate rate, it also calculates ETA when the full size of the origin file is known.

By default, `progress` only reports the disk write operations being done by coreutils programs- `cp`, `mv`, `dd`, `tar`, `cat`, etc. The full list is available in the [source code here](https://github.com/Xfennec/progress/blob/master/progress.c#L60). Thankfully, options are provided to extend the list of programs monitored by `progress`.

To illustrate this, let's set up a download using `curl` that we want to monitor. I'll be using the same large files which were created in the script above.

In one terminal window,

```
$ python3 -m http.server
Serving HTTP on 0.0.0.0 port 8000 (http://0.0.0.0:8000/) ...
```

and in another,

```
$ cd /tmp/progress_test
$ curl --output somefile http://localhost:8080/File_1000000
```

While this `curl` command is running, we can observe the file write operation happening by watching the `curl` process name with `-c`:

```
$ progress -c curl
[ 2418] curl /tmp/progress_test/somefile
        100.0% (557.0 MiB / 557.0 MiB)

```

The `-c`/`--command` flag tells `progress` to only watch for that command name when scanning `/proc` for commands that are copying data. This essentially replaces the default list of commands referenced above. If you'd instead like to include other commands while still scanning for all the default commands, pass `-a`/`--additional-command`, which can be given multiple times.

If you want to bypass this "scanning through /proc" behavior, you can directly pass process IDs with the `-p`/`--pid` argument.

## tl;dr

(re)Summarize why this tool is interesting and worth using.

:information_source: **Interested in seeing more posts like this?** Subscribe to the [cli.fan RSS feed](/posts/index.xml) linked here and in the top navigation bar.
