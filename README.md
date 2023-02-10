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

# Using `initializeCommand` portably

This repository has a dev container configuration that uses `initializeCommand`
and works on both Unix-like hosts (GNU/Linux and macOS) and Windows hosts, even
though Unix-like and Windows systems do not, in general, share any scripting
runtime.

Note that, while `initializeCommand` always runs in an environment associated
with the host and is never run inside the container, this environment is
*usually* still a Linux-based system. So this technique is [only
occasionally](#where-initializecommand-runs) needed.

## License

This repository is licensed under [0BSD](https://spdx.org/licenses/0BSD.html).
See [**`LICENSE`**](LICENSE).

## The technique

*This is a brief summary of the technique. For a fuller explanation, see
[`details.md`](details.md).*

Running a command like `./xyz` on Unix-like systems runs a file whose name is
exactly `xyz`. But on Windows, it will prefer files with extensions indicating
they are executable. So if you have a shell script with no extension, and a
batch file named similarly except for its `.cmd` extension, then the same
command in `devcontainer.json` will run the shell script or the batch file:
whichever one of them the host supports.

If you name the shell script `initialize` and the batch file `initialize.cmd`,
and you put them both directly in `.devcontainer/`, then this
`initializeCommand` in
[`.devcontainer/devcontainer.json`](.devcontainer/devcontainer.json) achieves
that:

```json
"initializeCommand": [".devcontainer/initialize"]
```

## Contents

The interesting files in this repository, besides this README, are:

- [`details.md`](details.md), which gives an in-depth explanation
- [`.devcontainer/devcontainer.json`](.devcontainer/devcontainer.json)
- [`.devcontainer/initialize`](.devcontainer/initialize), a shell script
- [`.devcontainer/initialize.cmd`](.devcontainer/initialize.cmd), a batch file

Of those files, all but [`details.md`](details.md) correspond to files you
would create in your repository if you apply the technique suggested here.
