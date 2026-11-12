# buckram

An MCP server that gives an AI agent a project's tools without a shell.

An agent that works through a shell writes text a shell then parses:
backslashes collapse, `$` expands, quotes nest, `&&` isn't there, encodings
change. buckram's tools take JSON instead, and run each program with its
arguments as a list, so what the agent writes arrives as it is.

Written in ['rain](https://github.com/) (a systems language for agents to
write), in one file, [buckram.rain](buckram.rain).

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
- **the project's tasks**, from its `buckram.json` (below): each a tool.

Each gives back the exit code, then the output and errors. Past
`output_limit` bytes (8192 by default) the middle is cut, at a character's
start, and the whole of it is kept in `output_folder/<tool>.txt`. A tool
whose program exits with a code other than 0, or runs past its timeout, is
an error result.

## A project's buckram.json

buckram reads `buckram.json` from the folder it's started in:

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
`timeout` is in seconds. A task can't be called `run`, `git` or `edit`.

Run real programs where you can: a `.bat` file goes through `cmd.exe`,
which parses its command line again.

## Building and using it

```
rain buckram.rain
```

makes `buckram.exe`. For Claude Code, add it to the project's `.mcp.json`:

```json
{
  "mcpServers": {
    "buckram": { "command": "c:/repos/buckram/buckram.exe" }
  }
}
```

It speaks MCP over standard input and output, a JSON-RPC message a line,
and writes only those there; what it has to say otherwise goes to
standard error.
