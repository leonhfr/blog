---
title: Help Message for Shell Scripts
description: Description
date: 2022-02-16
author: leon h
draft: false
favorite: false
# tags:
#   - tag1
---

Shell scripts are awesome. They are so useful to automate repetitive and boring work. The hardest thing about them is documentation though. How often have you written one, put it in the `bin` directory and forgot all about it? How cool would it be to have a help message for them?

<!--more-->

We could, of course, implement it with a bunch of echo calls. But there's a neat trick. I originally learned it in a blog post by [Egor Kovetskiy](https://github.com/kovetskiy), but it seems it's not available anymore. As it was very useful to me, I'm putting it out there again.

Add your help message as comments at the top of your file, right after the shebang.

```sh
#!/bin/bash
###
### my-script — does one thing well
###
### Usage:
###   my-script <input> <output>
###
### Options:
###   <input>   Input file to read.
###   <output>  Output file to write. Use '-' for stdout.
###   -h        Show this message.
```

Next, we need to get this message using `sed`.

```sh
sed -rn 's/^### ?//p' "$0"
```

What's happening here:

- `$0` is the filename of the file that is being executed;
- `-r` flag means using extended regular expressions;
- `-n` flag prevents `sed` from echoing each line to the standard output;
- `s` stands for substitution of the following pattern;
- `/` defines the start and end of the pattern;
- `^### ?` matches a string starting with `###` followed by an optional space;
- `//` defines the substitution string, here an empty string;
- `p` prints the result of the substitution.

Now, we just need to call this if an `-h` argument or no arguments are passed.

```sh
if [[ $# == 0 ]] || [[ "$1" == "-h" ]]; then
  sed -rn 's/^### ?//p' "$0"
  exit 0
fi
```

Hope it helps!
