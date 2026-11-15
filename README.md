# buckram

An MCP server that gives an AI agent a project's tools without a shell.

An agent that works through a shell writes text a shell then parses:
backslashes collapse, `$` expands, quotes nest, `&&` isn't there, encodings
change. buckram's tools take JSON instead, and run each program with its
arguments as a list, so what the agent writes arrives as it is.

An agent setting it up, or working on it, reads [AGENTS.md](AGENTS.md).

Written in 'rain, a systems language for agents to write: the stub,
[src/stub.rain](src/stub.rain); the tools, [src/core.rain](src/core.rain);
and its build, [build.rain](build.rain). Built, they're `bin/buckram.exe`
and `bin/buckram_core.exe` for Windows, and `bin/buckram` and
`bin/buckram_core` for Linux (x86-64), which are in the repository, so it
runs as it is downloaded: nothing needs building first.

## Tools

- **run**: a program (a path, or a name on the PATH), its arguments as a
  list, and if wanted the folder it starts in, its standard input and a
  timeout (600 seconds by default). A timeout ends what the program started
  too (on Windows, it runs in a job object of its own). git isn't run here,
  but by the git tool.
- **git**: any of git's commands (`operation`), with more arguments as a
  list, in the project or in `folder`. A commit's message is given as
  `message`, and goes to git as its input (`git commit -F -`), never on a
  command line. Before a command that can lose uncommitted work
  (`checkout`, `reset`, `restore`, `stash`, `clean`, `rebase`, ...: any but
  those that only look or only add, such as `status`, `log`, `add`,
  `commit`, `fetch`), the working tree is saved as a commit, untracked
  files too (not ignored ones), through a copy of the index, so neither the
  working tree nor the index changes. It's kept as
  `refs/buckram/saved/<when>`, for 30 days (older ones go as a new one is
  saved), and the result begins with it: `git restore
  --source=<commit> -- <path>` brings a file back. A command that makes a
  loss permanent (`reflog expire` or `delete`, `gc --prune`, `prune`, a
  forced push, a deleted remote branch) runs only with `"permanent":
  true`, which an agent is told needs the person's say-so.
- **edit**: exact replacements in files, all or none. Each edit's old text
  must be in its file exactly `count` times (1 by default); if any isn't,
  nothing is written. An empty old text makes a new file.
- **files**: list a folder (each entry's size, when it was last written,
  and its name; `all` for everything beneath it), copy or move a file or a
  whole folder, and delete files and folders, read-only ones included (as
  Perforce leaves them). It won't delete or move away the project's folder,
  one holding it, or a drive. A file a copy or move would replace that's
  in use (a running program, which can't be replaced but can be renamed)
  is moved into `old_exes` beside it first, where it runs on, and put back
  if the copy still fails; those there no longer in use are cleared out
  the next time.
- **processes**: list the running processes (those whose program's name
  contains `name`), or end them by program name or ID. buckram, and what
  started it (the agent's own client), are never ended.
- **the project's tasks**, from its `buckram.json` (below): each a tool.
  It runs in the project's folder, or in `folder` (a git worktree, say),
  where a program the task gives as a path (`build/tests.exe`) is that
  folder's: so a worktree's `build` and `test` are its own.
- **task**: runs a task by name, for one added since the client last
  listed the tools.
- **add_task** and **remove_task**: change the project's tasks, for one
  an agent finds it needs while it works. They're written into
  `buckram.json` (laid out to read, so a change shows in `git diff`).

Each call is logged, a JSON object a line, in
`output_folder/calls.<machine>.jsonl` (the machine and its system,
`dolly-windows` say, as two can share a project's folder: a VM's shared
folder, or WSL): when, the tool and its arguments, how long it took,
whether it was an error, and how big its result was. An error's line says
why (`"why"`): a program's exit code, or its timeout, and the first line
of its errors (or of its output, if it wrote none to standard error), or
the first line of buckram's own answer, a refusal say. It shows which jobs
an agent does often enough to be a task, or a tool of buckram's own.
[tools/calls.rain](tools/calls.rain) sums it up: `rain -run
c:/repos/buckram/tools/calls.rain [logs or folders] [-since:2026-11-14]`
(`build/buckram` by default) gives the calls and errors of each tool, each
program and each day, the commands made most often, what the errors said,
the slowest calls and the latest errors.

A tool refuses a parameter it doesn't have, naming those it has, and one
of the wrong type (a number in `arguments`, `"true"` for `permanent`, a
number for `folder`), saying what to give, rather than dropping it or
reading it as empty; a `null` is a parameter not given. `edit` refuses an
edit with a field it doesn't have, or without `file`, `old` or `new`.

Each gives back the exit code, then the output and errors. Past
`output_limit` bytes (8192 by default) the middle is cut, at a character's
start, and the whole of it is kept in `output_folder/<tool>.<machine>.txt`. A tool
whose program exits with a code other than 0, or runs past its timeout, is
an error result.

## A project's buckram.json

```json
{
  "output_limit": 8192,
  "output_folder": "build/buckram",
  "tasks": {
    "build": { "description": "Builds it.", "run": ["tools/build.exe"] },
    "test":  { "description": "Runs the tests.", "run": ["build/tests.exe"],
               "arguments": "list", "timeout": 900 }
  }
}
```

A task's `run` is the program and its arguments; a program given as a
path is the project's. It's one command for every system, or one for
each: `"run": {"windows": ["cmd", "/c", ".\\build.bat"], "linux":
["./build.sh"]}`. A task with no command for the system buckram runs on
isn't one of its tools there (the `task` tool names it, and which systems
it's for), so a project's folder shared between systems offers each its
own. `add_task` writes this system's command into a task that has one
for each, keeping the other's, and so into any task of a file whose tasks
give one for each (a task's one command then stays the other system's);
in any other file, the one command;
`remove_task` takes this system's out of a task that has one for each,
and otherwise the whole task. Its `arguments` is `"none"` (the default) or
`"list"`, for a tool that takes a list of strings to add after them. Its
`timeout` is in seconds (600 by default), and a call can give its own (a
number, or a string of one). A task can't have the name of one of
buckram's own tools.

Run real programs where you can: a `.bat` file goes through `cmd.exe`,
which parses its command line again.

## How it reloads

`bin/buckram.exe` is a stub, which the agent's harness starts and keeps
running. It answers `initialize` itself, and for any other message starts
`bin/buckram_core.exe` afresh, in the project's folder, gives it the message
as its standard input, and passes on its answer. So a rebuilt core, or a
changed `buckram.json`, is in use from the next message on: nothing needs
restarting. Nothing reaches the core on a command line.

When `buckram.json` or the core has changed since the last message, the
stub sends `notifications/tools/list_changed`, so the client lists the
tools again; it says so once a session starts too, as a client may keep an
old list.

## Using it

For Claude Code, add it to the project's `.mcp.json`, with the project's
folder as its argument:

```json
{
  "mcpServers": {
    "buckram": { "command": "c:/repos/buckram/bin/buckram.exe", "args": ["c:/path/to/project"] }
  }
}
```

On Linux the command is `bin/buckram` (`"command":
"/home/you/buckram/bin/buckram"`), which runs `bin/buckram_core` beside it.
It runs from a copy of itself in the machine's memory (`$XDG_RUNTIME_DIR`,
or `/tmp`), since `bin/` may be a shared folder that a build on Windows
replaces it in, and a running program whose file is replaced there dies.
Its tools are the same; a task's programs are Linux's (`sh`, `make`, a
project's own), and a project that's used on both keeps a `buckram.json`
each machine writes for itself.

## Building it

Only after changing `src/`, with the 'rain compiler:

```
rain build.rain
build.exe [path to rain.exe]
```

builds `bin/buckram.exe` and `bin/buckram_core.exe`, and the Linux ones,
`bin/buckram` and `bin/buckram_core` (with `-os:linux`, so it needs LLVM's
clang, as the compiler does for Linux). It's built on Windows, where the
compiler runs. Windows won't overwrite a running exe but will rename one,
so one in use is moved into `bin/old_exes` first, and runs on from there;
those no longer running are cleared out the next time. Commit the new
programs with the sources.

It speaks MCP over standard input and output, a JSON-RPC message a line,
and writes only those there; what it has to say otherwise goes to
standard error.
