---
title: "jo"
date: 2020-03-26T17:26:20-04:00
description: "jo: Construct JSON objects on the command-line"
draft: false
---

# jo

`jo` is a tool for creating JSON structures on the command-line. It converts `key=value` arguments into JSON objects and sequences into arrays (with the `-a` flag). `jo` is the perfect complement to a tool like [`jq`](https://stedolan.github.io/jq/) or [`fx`](https://github.com/antonmedv/fx); together they give you a full toolkit for working with JSON in shell scripts and ad-hoc on the command line.

![jo overview](/jo_main.png)

| Quick Facts | |
| ---- | ----------- |
| Version Control | git on [github](https://github.com/jpmens/jo) |
| Author | JP Mens ([jpmens](https://github.com/jpmens)) |
| Language | C |
| Binary Size | `128kb` |
| Version Reviewed | 1.3 |

## Installation

`jo` is distributed as a release tarball that you build and install yourself. Releases are available [here](https://github.com/jpmens/jo/releases), and instructions are [here](https://github.com/jpmens/jo#build-from-release-tarball).

It's also available in the following package managers:

| Platform | Installation Command |
| -------- | -------------------- |
| Ubuntu | `apt-get install jo` |
| Void Linux | `xbps-install -S jo` |
| HomeBrew | `brew install jo` |
| AUR | Install `jo` with your favorite AUR package manager |

## Usage

`jo`'s [manual](https://github.com/jpmens/jo/blob/master/jo.md) comes with a couple sample invocations that illustrate how to make JSON objects, but for this post I think a real-world example is a better illustration of how to use it.

For a simple usage of JSON, let's try to make a bash script that uses Github's API to upload a gist. The API takes JSON input in [the following format](https://developer.github.com/v3/gists/#example). Here's the example we'll try to recreate:

```javascript
{
  "description": "Hello World Examples",
  "public": true,
  "files": {
    "hello_world.rb": {
      "content": "class HelloWorld\n   def initialize(name)\n      @name = name.capitalize\n   end\n   def sayHi\n      puts \"Hello !\"\n   end\nend\n\nhello = HelloWorld.new(\"World\")\nhello.sayHi"
    }
  },
}
```

Just like the examples from the README, we can construct the top-level JSON object pretty easily:

```
$ jo -p description="Hello World Examples" public@true files=
{
   "description": "Hello World Examples",
   "public": true,
   "files": null
}
```

Here we're using the `-p` flag to pretty-print the resulting JSON. As the examples get more complex, I'll use [`jq`](https://stedolan.github.io/jq/) for the same functionality.

To replicate the target JSON's structure, we need to create nested objects in the `files` key. One way to do this is by using [command substitution](https://www.tldp.org/LDP/abs/html/commandsub.html):

```
$ jo -p description="Hello World Examples" public@true files=$(jo hello_world.rb=)
{
   "description": "Hello World Examples",
   "public": true,
   "files": {
      "hello_world.rb": null
   }
}
```

This works because if any value starts with `{`, `jo` interprets the rest of the value as a nested JSON expression. Once you get over a couple layers of nested objects, the parens remind me of LISP:

```
$ jo -p description="Hello World Examples" public@true files=$(jo hello_world.rb=$(jo content=foo))
{
   "description": "Hello World Examples",
   "public": true,
   "files": {
      "hello_world.rb": {
         "content": "foo"
      }
   }
}
```

Of course, the content is not `"foo"`, but the contents of `hello_world.rb`. `jo` can handle reading files like this for you with the `@` directive; `jo foo=@bar.txt` will construct a JSON object where the value of key `foo` is the contents of `bar.txt`.

```
$ jo -p content=@hello_world.rb
{
   "content": "class HelloWorld\n   def initialize(name)\n      @name = name.capitalize\n   end\n   def sayHi\n      puts \"Hello !\"\n   end\nend\n\nhello = HelloWorld.new(\"World\")\nhello.sayHi"
}
```

Let's put it all together to assemble the response:

```
$ jo -p description="Hello World Examples" public@true files=$(jo hello_world.rb=$(jo content=@hello_world.rb))
Argument `HelloWorld\n' is neither k=v nor k@v
Argument `def' is neither k=v nor k@v
Argument `initialize(name)\n' is neither k=v nor k@v
Argument `name.capitalize\n' is neither k=v nor k@v
Argument `end\n' is neither k=v nor k@v
Argument `def' is neither k=v nor k@v
Argument `sayHi\n' is neither k=v nor k@v
Argument `puts' is neither k=v nor k@v
Argument `\"Hello' is neither k=v nor k@v
Argument `!\"\n' is neither k=v nor k@v
Argument `end\nend\n\nhello' is neither k=v nor k@v
Argument `HelloWorld.new(\"World\")\nhello.sayHi"}' is neither k=v nor k@v
{
   "description": "Hello World Examples",
   "public": true,
   "files": {
      "hello_world.rb": "{\"content\":\"class",
      "": false,
      "": null,
      "": null
   }
}
```

Well that didn't work at all! Now we see the downside of using multiple layers of command substitution for composing these objects: shell quoting! Remember we're working in shell here, not a programming language with consistent whitespace handling :smile:

Thankfully, `jo` has a feature for this. In addition to reading a file as plain text with `@`, you can read a JSON file with `:`. Dumping an inner JSON object to a file, then re-reading it is one option. Another option is to use [process substitution](https://www.tldp.org/LDP/abs/html/process-sub.html) (aka the penguin operator) to connect the stdin and stdout of multiple commands.

```
$ cat test.txt
File contents
with whitespace
$ jo -p foo=:<(jo bar=:<(jo baz=@test.txt) qux=@test.txt)
{
  "foo": {
    "bar": {
      "baz": "File contents\nwith whitespace"
    },
    "qux": "File contents\nwith whitespace"
  }
}
```

Using process substitution lets `jo` read the nested JSON document as a file rather than additional arguments to the command. Since stdin and stdout are files instead of strings passed as arguments, whitespace is handled correctly.

```
$ jo -p description="Hello World Examples" public@true files=:<(jo hello_world.rb=:<(jo content=@hello_world.rb))
{
   "description": "Hello World Examples",
   "public": true,
   "files": {
      "hello_world.rb": {
         "content": "class HelloWorld\n   def initialize(name)\n      @name = name.capitalize\n   end\n   def sayHi\n      puts \"Hello !\"\n   end\nend\n\nhello = HelloWorld.new(\"World\")\nhello.sayHi"
      }
   }
}
```

Now that we can properly construct this JSON document, let's use `curl` to hit the API:

```
$ curl -XPOST 'https://api.github.com/gists' -u "delucks:${GITHUB_TOKEN}" -d "$(jo description="Hello World Examples" public@true files=:<(jo hello_world.rb=:<(jo content=@hello_world.rb)))" | jq -r .html_url
https://gist.github.com/delucks/31295c493f9d3ea3906869e97a8192ef
```

Great! To generalize this a bit, we could turn it into a shell function that uploads public gists of a single file.

```bash
gistit() {
  curl -XPOST 'https://api.github.com/gists' -u "$(whoami):${GITHUB_TOKEN}" -d "$(jo description="${2:-Gist uploaded at $(date)}" public@true files=:<(jo $(basename $1)=:<(jo content=@${1})))" | jq -r .html_url
}
```

As you can see, `jo` lets you construct JSON documents quickly and legibly compared with concatenating and formatting strings alone.

Let's try a more involved example. Recently, systemd added support for a service called [`systemd-homed`](https://systemd.io/HOME_DIRECTORY/), which manages home directories of users for portability between different systems or drives. It is configured by JSON files within a directory `~/.identity` that are formatted according to [this specification](https://systemd.io/USER_RECORD/). Since this feature is going to be used widely across many different environments, creating these configuration files within a shell context is useful. Let's replicate the most involved example from this page using the following `jo` invocation:

```
$ DRIVE_UUID=$(uuidgen | tr -d -- -)
$ jo \
  autoLogin@true \
  binding=:<(jo "$DRIVE_UUID"=:<(jo fileSystemType=ext4 fileSystemUuid=$(uuidgen) gid=60232 homeDirectory=/home/delucks imagePath=/home/delucks.home luksCipher=aes luksCipherMode=xts-plain64 luksUuid=$(uuidgen) luksVolumeKeySize=32 partitionUuid=$(uuidgen) storage=luks uid=60232)) \
  status=:<(jo "$DRIVE_UUID"=:<(jo goodAuthenticationCounter=16 lastGoodAuthenticationUSec=$(date +%s%N) rateLimitBeginUSec=1566309342340723 rateLimitCount=1 state=inactive service=io.systemd.Home diskSize=161118667776 diskCeiling=190371729408 diskFloor=5242880 signedLocally@true)) \
  disposition=regular \
  enforcePasswordPolicy@false \
  lastChangeUSec=$(date +%s%N) \
  memberOf=:<(echo wheel | jo -a) \
  privileged=:<(jo hashedPassword=$(openssl passwd -6 -salt xyz Nonsense | jo -a)) \
  userName=delucks \
  locked@false | jq -S .
```

This produces the following JSON document, which is a fully formed example of how to configure `systemd-homed`:

```javascript
{
  "autoLogin": true,
  "binding": {
    "9a94dd8cc69d407f8dc6d73d026a5beb": {
      "fileSystemType": "ext4",
      "fileSystemUuid": "330a64b4-24e5-4015-a28b-db3a56b43f2c",
      "gid": 60232,
      "homeDirectory": "/home/delucks",
      "imagePath": "/home/delucks.home",
      "luksCipher": "aes",
      "luksCipherMode": "xts-plain64",
      "luksUuid": "f41cead2-bfba-41e2-9c16-15eff8e5d0d0",
      "luksVolumeKeySize": 32,
      "partitionUuid": "b4b5b509-c74e-4c4a-b2f5-b35272e652c1",
      "storage": "luks",
      "uid": 60232
    }
  },
  "disposition": "regular",
  "enforcePasswordPolicy": false,
  "lastChangeUSec": 1585257325374165000,
  "locked": false,
  "memberOf": [
    "wheel"
  ],
  "privileged": {
    "hashedPassword": [
      "$6$xyz$YqRpzp3ROcNuHgzLDUAv.CcOW0DXV8paXX8eW0002NWe7.ke.4QT0c3tpHzktZh26Mr0ZyOJJ1wc.JIfGGT/h/"
    ]
  },
  "status": {
    "9a94dd8cc69d407f8dc6d73d026a5beb": {
      "diskCeiling": 190371729408,
      "diskFloor": 5242880,
      "diskSize": 161118667776,
      "goodAuthenticationCounter": 16,
      "lastGoodAuthenticationUSec": 1585257325395444000,
      "rateLimitBeginUSec": 1566309342340723,
      "rateLimitCount": 1,
      "service": "io.systemd.Home",
      "signedLocally": true,
      "state": "inactive"
    }
  },
  "userName": "delucks"
}
```

Some of the parts of this example are clearly contrived; this generates new disk UUIDs every time it's run and uses a password of "Nonsense" - but it illustrates that you're a step away from automating creation of these JSON files using `jo`. It's a simple matter to substitute real commands for measuring disk quota, gathering partition UUIDs, etc.

## tl;dr

There are already a number of supporting utilities for _reading_ and _parsing_ JSON data, but `jo` allows the creation of JSON documents to be done within the shell as well. For simple JSON structures, it cleans up the formatting within your commands and adds features like reading text files and nested documents. For more complex structures, `jo` bridges the gap between ad-hoc string construction and a programming language with a full JSON implementation.

:information_source: **Interested in seeing more posts like this?** Subscribe to the [cli.fan RSS feed](/posts/index.xml) linked here and in the top navigation bar.
