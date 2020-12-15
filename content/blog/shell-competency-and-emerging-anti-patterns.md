---
title: "Shell Competency and Emerging Anti-Patterns"
date: 2020-12-14T14:00-06:00
tags:
    - devops
    - linux
    - shell
---

Throughout my career, I've crossed paths with quite a few developers whose
chief expertise has run the gamut of programming languages: C, Python,
Java, Ruby, JavaScript, and so on. The programming language(s) in which you
build expertise undoubtedly influence the way you think about problems and
what solutions you're likely to reach for, and they do so in a big way.
The common thread I've noticed, though, is that developers on average lack
competency in their operating system shell. That lack of competency often
manifests as an *aversion* to the shell.

You may see that a developer prefers to use point-and-click interfaces
when a shell can get the job done more quickly and with exponentially
more flexibility. You might find that they write scripts in their native
language rather than struggle with the intricacies of `bash`. At one of
my previous workplaces, I uncovered a 30 line Ruby script that could
have been replaced by one pipeline in `bash`. Yes, a single pipeline.

This shell aversion is a real shame, because the shell can be a very powerful
tool in one's technical arsenal. The shell is _the_ textual interface to your
computer.  Regardless of familiarity, every developer will be forced to use a
shell in some ongoing capacity. That use may be to invoke `git`, grab a quick
snapshot of what is happening in the cloud with `aws`, or start a container
with `docker` or `docker-compose`. The uses are there, and they are many.

Because my experience has largely been infrastructure focused, the shell is more
often a sensible tool for me to reach for than it is for other developers.
Consequently, I learned many hard lessons about shell, best practices,
patterns, and anti-patterns.

And that brings us to the topic at hand today. A friend sent me a blog post
claiming to implement a "safe" template for bash. Just insert your shell
into the provided blank. It's a fairly hefty template too, weighing in at
just under 100 lines.

Complements of [Better Dev.blog](https://betterdev.blog/minimal-safe-bash-script-template/),
the template follows:

{{< highlight bash "linenos=table" >}}
#!/usr/bin/env bash

set -Eeuo pipefail

cd "$(dirname "${BASH_SOURCE[0]}")" >/dev/null 2>&1

trap cleanup SIGINT SIGTERM ERR EXIT

usage() {
  cat <<EOF
Usage: $(basename "$0") [-h] [-v] [-f] -p param_value arg1 [arg2...]

Script description here.

Available options:

-h, --help      Print this help and exit
-v, --verbose   Print script debug info
-f, --flag      Some flag description
-p, --param     Some param description
EOF
  exit
}

cleanup() {
  trap - SIGINT SIGTERM ERR EXIT
  # script cleanup here
}

setup_colors() {
  if [[ -t 2 ]] && [[ -z "${NO_COLOR-}" ]] && [[ "${TERM-}" != "dumb" ]]; then
    NOCOLOR='\033[0m' RED='\033[0;31m' GREEN='\033[0;32m' ORANGE='\033[0;33m' BLUE='\033[0;34m' PURPLE='\033[0;35m' CYAN='\033[0;36m' YELLOW='\033[1;33m'
  else
    NOCOLOR='' RED='' GREEN='' ORANGE='' BLUE='' PURPLE='' CYAN='' YELLOW=''
  fi
}

msg() {
  echo >&2 -e "${1-}"
}

die() {
  local msg=$1
  local code=${2-1} # default exit status 1
  msg "$msg"
  exit "$code"
}

parse_params() {
  # default values of variables set from params
  flag=0
  param=''

  while :; do
    case "${1-}" in
    -h | --help)
      usage
      ;;
    -v | --verbose)
      set -x
      ;;
    --no-color)
      NO_COLOR=1
      ;;
    -f | --flag) # example flag
      flag=1
      ;;
    -p | --param) # example named parameter
      param="${2-}"
      shift
      ;;
    -?*)
      die "Unknown option: $1"
      ;;
    *)
      break
      ;;
    esac
    shift
  done

  args=("$@")

  # check required params and arguments
  [[ -z "${param-}" ]] && die "Missing required parameter: param"
  [[ ${#args[@]} -eq 0 ]] && die "Missing script arguments"

  return 0
}

parse_params "$@"
setup_colors

# script logic here

msg "${RED}Read parameters:${NOCOLOR}"
msg "- flag: ${flag}"
msg "- param: ${param}"
msg "- arguments: ${args[*]-}"
{{< /highlight >}}

I won't sugar coat it. There's a *lot* wrong with this. Let's start
from the top.

### The Shebang

{{< highlight bash "linenos=table" >}}
#!/usr/bin/env bash
{{< /highlight >}}

The classic shebang. In case you don't know what it is, here's the quick
rundown: generally, when you execute a file on your computer, it's a raw
binary that your operating system understands and can execute natively.
Being that scripts aren't binaries, a file starting with `#!` signals
that instead, the command that follows should be executed and the the
file path passed to the interpreter.

In this case, we've got `env` searching a user's `PATH` and executing
`bash`. Proponents of this method would tell you that it smooths over
file system hierarchy differences between machines. That's technically
true.

But there's a big problem here: if you're at the level of experience
where you're copy/pasting a template, your `bash` script isn't
going to run on any machine other than the one you wrote it on (or
other very similar systems where `bash` can be found at the same
place). Using `#!/usr/bin/env bash` as the shebang gives the
*misleading* impression that you can just take the script and
run it anywhere. And that's just not true unless your script does
almost nothing of interest, or you actually understand the differences
between various platforms and can account for them when writing
your script. Furthermore, as the author, you probably originally
wrote the script with the system-wide `bash` in mind - so why
shouldn't you just use that one?

In short, don't misrepresent your script from the outset. Just use
`#!/bin/bash` unless you can actually handle running in another
environment.

### The `set` Smorgasbord

{{< highlight bash "linenos=table" >}}
set -Eeuo pipefail
{{< /highlight >}}
