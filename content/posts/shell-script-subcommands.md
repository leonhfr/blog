---
title: Simple shell script subcommands
description: How to add git-style subcommands to shell scripts with self-documenting help messages.
date: 2022-10-01
draft: false
tags:
  - shell
  - bash
---

Shell scripts are awesome, [as we know](./shell-script-help.md). Now, we have nice shell scripts that self document. But if you're anything like me, you have lots of them in your `~/bin` or wherever you put them. One reason is that each script can do one thing and one thing only. Wouldn't it be nice if we could group different functionalities in the same file?

<!--more-->

Some CLIs already do that with subcommands. Think `git add` or `go get`. We're going to check how to simply have subcommands with shell scripts.

First, we put each functionality in its own function. Each function should be prefixed `sub_`. Of course, we have the documentation at the top and a help function to parse and print the documentation. Note that each function also has a comment.

```sh
#!/bin/bash
###
### my-script — does several things well
###
### Usage: my-script [options] <subcommand>
###
### Options:
###   -h,--help: Show this message.
###
### Subcommands:

### help: show this message
sub_help() {
  sed -rn 's/^### ?//p' "$0"
}

### foo: do some the foo thing
sub_foo() {
    echo "Running the foo subcommand"
}

### bar: do the bar thing, which is related but different to the foo thing
sub_bar() {
    echo "Running the bar subcommand"
    echo "We can use first argument with '$1'"
    echo "The second argument is '$2', and so on"
}
```

Then, at the end of the script, we put this piece of magic:

```sh
SCRIPT_NAME=$(basename $0)

SUBCOMMAND=$1
case $SUBCOMMAND in
  "" | "-h" | "--help")
    sub_help
    ;;
  *)
    shift
    sub_${SUBCOMMAND} $@
    if [ $? = 127 ]; then
      echo "Error: '$SUBCOMMAND' is not a known subcommand." >&2
      echo "       Run '$SCRIPT_NAME --help' for a list of known subcommands." >&2
      exit 1
    fi
    ;;
esac
```

What's happening here? Some notes:

- we store the name of the script with `basename`;
- we use a bash [case statement](https://linuxize.com/post/bash-case-statement/);
- we check the first argument, the subcommand, if it's one of `-h`, `--help`, or empty, we print the help and exit;
- otherwise (see the `*)` which is the default case), we:
  - use the `shift` command to shift positional parameters to the left, so command name which was in `$1` is placed at `$0` instead, and so on for other arguments;
  - we execute the subcommand function with the repositioned arguments `$@`;
  - if the subcommand doesn't exist, we print an error message using the script name.

Now the cool thing about this is that not only can we add as many subcommands as we like, but the help message also gets updated with the subcommand comments, so the script is still self documenting.

Printing the help:

```sh
$ my-script --help
my-script — does several things well

Usage: my-script [options] <subcommand>

Options:
  -h,--help: Show this message.

Subcommands:
help: show this message
foo: do the foo thing
bar: do the bar thing, which is related but different to the foo thing
```

Running the bar subcommand:

```sh
$ my-script bar hello world
Running the bar subcommand
We can use first argument with 'hello'
The second argument is 'world', and so on
```

Command that doesn't exist:

```sh
$ my-script hello
Error: 'hello' is not a known subcommand.
       Run 'my-script --help' for a list of known subcommands.
```
