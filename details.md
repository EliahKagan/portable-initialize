<!--
  Copyright (c) 2023 Eliah Kagan

  Permission to use, copy, modify, and/or distribute this software for any
  purpose with or without fee is hereby granted.

  THE SOFTWARE IS PROVIDED "AS IS" AND THE AUTHOR DISCLAIMS ALL WARRANTIES WITH
  REGARD TO THIS SOFTWARE INCLUDING ALL IMPLIED WARRANTIES OF MERCHANTABILITY
  AND FITNESS. IN NO EVENT SHALL THE AUTHOR BE LIABLE FOR ANY SPECIAL, DIRECT,
  INDIRECT, OR CONSEQUENTIAL DAMAGES OR ANY DAMAGES WHATSOEVER RESULTING FROM
  LOSS OF USE, DATA OR PROFITS, WHETHER IN AN ACTION OF CONTRACT, NEGLIGENCE OR
  OTHER TORTIOUS ACTION, ARISING OUT OF OR IN CONNECTION WITH THE USE OR
  PERFORMANCE OF THIS SOFTWARE.
-->

# Using `initializeCommand` portably - Details

## Where `initializeCommand` runs

An `initializeCommand`, if defined, is run on the host, in the environment
where the repository's files reside.

### Docker on Windows uses a WSL system or VM as the "host"

On Windows, Docker uses a Linux-based system to do its work: depending which
backend Docker is configured to use, this is either a system in WSL 2, or a
full virtual machine run directly Hyper-V. When you clone a repository into a
container on Windows, the files are created in storage associated with this
Linux-based environment. So `initializeCommand` runs in the Linux-based system
Docker uses to do its work, and it is unnecessary for it to be able to run on
Windows.

### But Windows is sometimes the "host" for `initializeCommand`

The situation when `initializeCommand` runs on Windows is when you are using
Windows (*not* including a WSL system in Windows), and you reopen a local
folder in the container, rather than cloning the repository in a container.
This should usually be avoided, because except on GNU/Linux hosts, mapping a
local folder into a Docker container incurs considerable I/O slowdown. However,
using `initializeCommand` will often outright break that use case on Windows,
preventing the dev container from being created using a local folder, and
giving a potentially confusing error message. This may be worth avoiding.

## Initial consideration: string vs. array

When `initializeCommand` is used to run a script, it usually looks like one of:

```json
"initializeCommand": ".devcontainer/initialize"
```

```json
"initializeCommand": [".devcontainer/initialize"]
```

The first way, with a the string `".devcontainer/initialize"`, runs the command
`".devcontainer/initialize"` in a shell. The second way, with the one element
array `[".devcontainer/initialize"]`, executes a file indicated by
`.devcontainer/initialize` with no command-line arguments. (Subsequent elements
of the array, if present, would be passed as command-line arguments to
`.devcontainer/initialize`.)

The difference between the first and second way is not whether a shell is
*ever* used, but just whether the command `.devcontainer/initialize` is
*itself* parsed and interpreted by a shell even before any file it indicates is
opened and run. For simplicity, and to avoid another layer of indirection whose
behavior varies across host systems, the latter case is considered and
suggested here.

## Shown here: Similarly named shell script and batch file

Although windows uses `\` rather than `/` as its primary path separator, it
also supports `/`. Although not all Windows *utilities* recognize `/`, and some
ways of using `/` in user interfaces (especially if intermixed with `\`) can
result in extremely confusing user experiences, neither is a problem here.

Thus `.devcontainer/initialize` is a path, and its purpose is to indicate a
file to execute, and *the path itself* has the same meaning on Unix-like and
Windows systems. So it intuitively feels like the file
`.devcontainer/initialize`, at least if present, would always itself be the
file that path indicates. That is the case on Unix-like systems, but not on
Windows systems.

Windows makes heavier use of file extensions than Unix-like systems:

- On a Unix-like system, text files are made executable by setting executable
  permissions, and the appropriate interpreter is specified with [a leading
  `#!` line](https://en.wikipedia.org/wiki/Shebang_(Unix)), which allows a
  later version of the file to use a different interpreter (or even be written
  in a totally different language) without requiring any changes at the call
  site.
- On Windows, only a few kinds of text executables are on par with binary
  executables. Both text and binary executables are named with an extension to
  tell the system that they are executable, and how they are to be run. Most
  executable binaries are suffixed `.exe`, while most batch files are suffixed
  `.cmd`. Other kinds of scripts than batch files are used on Windows, but the
  interpreter is still selected based on file extension, by looking up what
  program should be used *to open files* with that extension. **When the
  extension is omitted, the system checks for files named the same way but with
  one a small handful of extensions.** This includes `.exe` and `.cmd`.

That sometimes leads to confusion. In this case, however, we can use it to
ensure well-defined `initializeCommand` behavior across the systems, with
whatever customizations we need.

***FIXME: Write the rest of this.***

## Alternative: Do you need it at all?

The simplest way to avoid an unportable `initializeCommand` is to not use
`initializeCommand` at all. `initializeCommand` runs on the host, and the
benefit of a dev container is isolation and reproducibility, allowing an
environment to be spun up and to work in the same way on any host. Doing more
work on the host, instead of inside the container, tends to weaken these
advantages. If you can do the work inside the container instead, you should.
This can be achieved by using one of the other [lifecycle
scripts](https://containers.dev/implementors/json_reference/#lifecycle-scripts)
instead, such as `onCreateCommand`.

## Alternative: Very simple commands

If you must have an `initializeCommand`, but it is simple and is guaranteed to
work on all systems, then nothing special has to be done. This is rare, but it
might be worth considering before doing something that involves more code.
