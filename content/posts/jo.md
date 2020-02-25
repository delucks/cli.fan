---
title: "jo"
date: 2020-01-07T22:26:37-05:00
description: "jo: Construct JSON objects on the command-line"
draft: true
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

Whaaats all this? Now we see the downside of using command substitution for composing together these objects: shell quoting! Remember we're working in shell here, not a programming language that understands whitespace.

Thankfully, `jo` has a feature for this. In addition to reading a file as plain text with `@`, you can read a JSON file with `:`. Dumping an inner JSON object to a file, then re-reading it is one option. Another option is to use [process substitution](https://www.tldp.org/LDP/abs/html/process-sub.html), which connects the stdin and stdout of multiple commands. Since stdin/stdout are files, we can get `jo` to read a nested JSON document as a file rather than arguments to the outer `jo` invocation, correctly handling whitespace.

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

That would be a nice little shell function:

```bash
gistit() {
  curl -XPOST 'https://api.github.com/gists' -u "$(whoami):${GITHUB_TOKEN}" -d "$(jo description="${2:-Gist uploaded at $(date)}" public@true files=:<(jo $(basename $1)=:<(jo content=@${1})))" | jq -r .html_url
}
```

As you can see, `jo` makes this kind of work pretty easy compared to manually assembling the JSON structure as a string.

Let's try a more involved example. If 

## tl;dr

(re)Summarize why this tool is interesting and worth using.

:information_source: **Interested in seeing more posts like this?** Subscribe to the [cli.fan RSS feed](/posts/index.xml) linked here and in the top navigation bar.
