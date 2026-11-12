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
and `bin/buckram_core.exe`, which are in the repository, so it runs as it
is downloaded: nothing needs building first.

## Tools

- **run**: a program (a path, or a name on the PATH), its arguments as a
  list, and if wanted the folder it starts in, its standard input and a
  timeout (600 seconds by default).
- **git**: `status`, `diff`, `log`, `show`, `add` and `commit`, with more
  arguments as a list. A commit's message is given as `message`, and goes
  to git as its input (`git commit -F -`), never on a command line. Nothing
  that rewrites or discards history or work is offered.
- **edit**: exact replacements in files, all or none. Each edit's old text
  must be in its file exactly `count` times (1 by default); if any isn't,
  nothing is written. An empty old text makes a new file.
- **files**: list a folder (each entry's size, when it was last written,
  and its name; `all` for everything beneath it), copy or move a file or a
  whole folder, and delete files and folders, read-only ones included (as
  Perforce leaves them). It won't delete or move away the project's folder,
  one holding it, or a drive.
- **processes**: list the running processes (those whose program's name
  contains `name`), or end them by program name or ID. buckram, and what
  started it (the agent's own client), are never ended.
- **the project's tasks**, from its `buckram.json` (below): each a tool.
- **task**: runs a task by name, for one added since the client last
  listed the tools.
- **add_task** and **remove_task**: change the project's tasks, for one
  an agent finds it needs while it works. They're written into
  `buckram.json` (laid out to read, so a change shows in `git diff`).

Each call is logged, a JSON object a line, in `output_folder/calls.jsonl`:
when, the tool and its arguments, how long it took, whether it was an
error, and how big its result was. It shows which jobs an agent does often
enough to be a task, or a tool of buckram's own.

Each gives back the exit code, then the output and errors. Past
`output_limit` bytes (8192 by default) the middle is cut, at a character's
start, and the whole of it is kept in `output_folder/<tool>.txt`. A tool
whose program exits with a code other than 0, or runs past its timeout, is
an error result.

## A project's buckram.json

```json
{
  "output_limit": 8192,
  "output_folder": "build/buckram",
  "tasks": {
    "build": { "description": "Builds it.", "run": ["python", "tools/dev.py", "build"] },
    "test":  { "description": "Runs the tests.", "run": ["python", "tools/dev.py", "test"],
               "arguments": "list", "timeout": 900 }
  }
}
```

A task's `run` is the program and its arguments; a program given as a
path is the project's. Its `arguments` is `"none"` (the default) or
`"list"`, for a tool that takes a list of strings to add after them. Its
`timeout` is in seconds. A task can't have the name of one of buckram's
own tools.

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

## Building it

Only after changing `src/`, with the 'rain compiler:

```
rain build.rain
build.exe [path to rain.exe]
```

builds `bin/buckram.exe` and `bin/buckram_core.exe`. Windows won't
overwrite a running exe but will rename one, so one in use is moved into
`bin/old_exes` first, and runs on from there; those no longer running are
cleared out the next time. Commit the new exes with the sources.

It speaks MCP over standard input and output, a JSON-RPC message a line,
and writes only those there; what it has to say otherwise goes to
standard error.
