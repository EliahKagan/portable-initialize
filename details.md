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

An
[`initializeCommand`](https://containers.dev/implementors/json_reference/#lifecycle-scripts),
if defined, is run on the host, in the environment where the repository's files
reside.

### Docker on Windows uses a WSL system or VM as the "host"

On Windows, [Docker](https://www.docker.com/) uses a Linux-based system to do
its work. Depending [which backend Docker is configured to
use](https://docs.docker.com/desktop/windows/wsl/), this is either a system in
[WSL
2](https://learn.microsoft.com/en-us/windows/wsl/compare-versions#whats-new-in-wsl-2),
or a full virtual machine run directly in
[Hyper-V](https://learn.microsoft.com/en-us/virtualization/hyper-v-on-windows/about/).

When you clone a repository into a container on Windows, the files are created
in storage associated with this Linux-based environment. So `initializeCommand`
runs in the Linux-based system Docker uses to do its work, and it is
unnecessary for it to be able to run on Windows.

### But Windows is sometimes the "host" for `initializeCommand`

The situation when `initializeCommand` runs on Windows is when you are using
Windows (*not* including a WSL system in Windows), and you [reopen a local
folder in the
container](https://code.visualstudio.com/docs/devcontainers/containers#_quick-start-open-an-existing-folder-in-a-container),
rather than [cloning the repository in a
container](https://code.visualstudio.com/docs/devcontainers/containers#_quick-start-open-a-git-repository-or-github-pr-in-an-isolated-container-volume).

This should usually be avoided, because except on GNU/Linux hosts, mapping a
local folder into a Docker container incurs [considerable I/O
slowdown](https://code.visualstudio.com/remote/advancedcontainers/improve-performance).
However, using `initializeCommand` will often outright break that use case on
Windows, preventing the dev container from being created using a local folder,
and giving a potentially confusing error message. This may be worth preventing.

## Initial consideration: string vs. array

When `initializeCommand` is used to run a script, it
[usually](https://containers.dev/implementors/json_reference/#formatting-string-vs-array-properties)
looks like one of:

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

### What file `.devcontainer/initialize` executes

Although windows uses `\` rather than `/` as its primary path separator, it
also supports `/`. Although not all Windows *utilities* recognize `/`, and some
ways of using `/` in user interfaces (especially if intermixed with `\`) can
result in extremely confusing user experiences, neither is a problem here.

Thus `.devcontainer/initialize` is a path, and its purpose is to indicate a
file to execute, and *the path itself* has the same meaning on Unix-like and
Windows systems. So it intuitively feels like the file
`.devcontainer/initialize`, at least if present, would always itself be the
file that path indicates *to run*. That is the case on Unix-like systems, but
not on Windows systems.

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
  any of a small handful of extensions added.** This includes `.exe` and
  `.cmd`.

That sometimes leads to confusion. In this case, however, we can use it to
ensure well-defined `initializeCommand` behavior across the systems, with
whatever customizations we need.

### `initialize` and `initialize.cmd` in this repository

The [`.devcontainer/`](`.devcontainer/`) directory of this repository contains
a shell script, `initialize`, and a batch file, `initialize.cmd`.

Even though `.devcontainer/initialize` exists, running that as a command on
Windows runs `.devcontainer/initialize.cmd` instead.

### Why use a batch file?

Batch files are [no longer the most recommended
way](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/windows-commands#command-line-shells)
to write scripts on Windows. But you should probably use a batch file for this,
even if all it does is delegate to some other script for its functionality.
Furthermore, although other kinds of scripts are often more powerful and more
pleasant to write, they also have limitations that may may preclude their use.

The major alternatives on Windows are:

#### Windows PowerShell

Every version of Windows that is currently supported has Windows
[PowerShell](https://learn.microsoft.com/en-us/powershell/) (`powershell.exe`).
By default, only scripts that originated on the local machine or that are
digitally signed by their authors will run. Although it is common to change
that setting, it cannot be assumed changed if compatibility with all Windows
systems (running currently supported versions) is a goal. Furthermore, minimal
host configuration is typically a goal of using dev containers. So requiring
users to change this setting is likely undesirable, even aside from any
security concerns.

Files with `.ps1` extensions are associated with Windows PowerShell by default.
But unlike with the `.cmd` extension for batch files, a file with a `.ps1`
extension will not be selected automatically when attempting to run a path with
no explicit file extension. Starting in a batch file and chaining into a
separate Windows PowerShell script can be done to get around this.

#### Cross-platform PowerShell

The cross-platform [PowerShell](https://learn.microsoft.com/en-us/powershell/)
(formerly called PowerShell Core) is often, but not always present. When
present on Windows, it is provided by `pwsh.exe`.

This is a nicer shell to work in than Windows PowerShell. For example, text
encoding defaults to UTF-8. However, it has the same issue with limitations on
script execution.

The cross-platform PowerShell is not the best choice for a script that must be
able to run on any Windows system, since Windows PowerShell is the only
PowerShell that is guaranteed to be present on Windows.

#### Windows Management Instrumentation

The [Windows Management
Instrumentation](https://learn.microsoft.com/en-us/previous-versions/windows/desktop/wmi_v2/windows-management-infrastructure)
Command-Line Utility,
[`wmic.exe`](https://support.microsoft.com/en-us/topic/a-description-of-the-windows-management-instrumentation-wmi-command-line-utility-wmic-exe-f5c16751-3a83-49ee-030d-5092ce1a04bb),
provides powerful scripting features. It is available on most Windows systems,
but not all. It is [deprecated in Windows 10 21H1 and not automatically
installed in Windows
11](https://www.elevenforum.com/t/add-or-remove-wmic-command-feature-in-windows-11.5119/).
This is [for security
reasons](https://www.bleepingcomputer.com/news/microsoft/microsoft-starts-killing-off-wmic-in-windows-will-thwart-attacks/).

Since some current and most future Windows systems will probably not have
`wmic.exe`, it is not a good choice for tasks that need to be able to run on
any Windows system.

#### Windows Script Host

Various other scripting languages are available, at least on some systems, in
Windows, using the [Windows Script
Host](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2003/cc738350(v=ws.10)).
Support for two languages,
[VBScript](https://learn.microsoft.com/en-us/previous-versions/t0aew7h6(v=vs.85))
and
[JScript](https://learn.microsoft.com/en-us/previous-versions/hbxc2t98(v=vs.85)),
is [available by
default](https://en.wikipedia.org/wiki/Windows_Script_Host#Available_scripting_engines).

*Forthcoming: Considerations for using the Windows Script Host for
`initializeCommand`.*

#### Unix-style shells

Unix-style shells and supporting environments, such as
[Bash](https://www.gnu.org/software/bash/), are available on Windows through a
variety of projects, such [MSYS2](https://www.msys2.org/) and
[Cygwin](https://cygwin.com/), as well as via
[WSL](https://learn.microsoft.com/en-us/windows/wsl/about).

None of this is guaranteed to be available. Other than WSL, none is usually
available. WSL is not always available, and relying on an *additional* WSL
system for dev containers, which are Docker-based, could be confusing and, on
some systems, lead to excessive resource usage.

## Alternative: Do you need it at all?

The simplest way to avoid an unportable `initializeCommand` is to not use
`initializeCommand` at all.

`initializeCommand` runs on the host, and the benefit of a dev container is
isolation and reproducibility, allowing an environment to be spun up and to
work in the same way on any host. Doing more work on the host, instead of
inside the container, tends to weaken these advantages. If you can do the work
inside the container instead, you should. This can be achieved by using one of
the other [lifecycle
scripts](https://containers.dev/implementors/json_reference/#lifecycle-scripts)
instead, such as `onCreateCommand`.

## Alternative: Very simple commands

If you must have an `initializeCommand`, but it is a single simple command that
is guaranteed to work the same way on all systems, then nothing special has to
be done. This is rare, but it might be worth considering before doing something
that involves more code.
