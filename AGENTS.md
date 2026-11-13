# Working with buckram

buckram is an MCP server that runs a project's programs for an agent, with
each program's arguments as a list, so no shell parses what the agent
writes. This file is for agents: setting buckram up in a project, using
it, and working on buckram itself. [README.md](README.md) describes it for
people.

## Using it

The server's instructions and each tool's description say what the tools
do; in short:

- **Prefer a task.** The project's `buckram.json` declares its build,
  test and other commands as tools of their own. Use those first.
- **Add a task for anything you'll run again.** `add_task` writes it into
  `buckram.json` (laid out to read, so it shows in `git diff`), and buckram
  tells the client its tools have changed: in Claude Code the task is a
  tool of its own from your next call (load it with ToolSearch if tools
  are deferred). `task` runs any task by name meanwhile. `remove_task`
  takes one out. A task's name is letters, digits and `_`.
- **`run` for a one-off**: a program (a path, or a name on the PATH), its
  arguments as a list, and if needed `folder`, `input` (all of its
  standard input) and `timeout` (seconds, 600 by default; it ends what the
  program started too). A task takes a `timeout` too, for one call. Not
  for git: `run` refuses it.
- **`git`** for any of git's commands: `operation` (`status`, `diff`,
  `log`, `add`, `commit`, `checkout`, `stash`, ...), more arguments as a
  list, and `folder` for a repository other than the project. Give a
  commit's message as `message`, exactly as it should be: it goes to git
  as its input. Before a command that can lose uncommitted work, buckram
  saves the working tree (untracked files too, not ignored ones) as a
  commit, `refs/buckram/saved/<when>`, without touching it or the index,
  and the result begins with its hash: `git restore --source=<hash> --
  <path>` brings a file back. A command that makes a loss permanent
  (`reflog expire`/`delete`, `gc --prune`, `prune`, a forced push or a
  deleted remote branch) runs only with `"permanent": true`: ask the
  person first, every time.
- **A parameter a tool hasn't** is refused, with the ones it has: read
  the error, don't drop the parameter blindly.
- **`files`** to list, copy, move or delete files and folders, rather than
  a program written for it: `operation` `list` (`path`, `all`), `copy` or
  `move` (`from`, `to`: a file or a whole folder), `delete` (`paths`; a
  folder goes with everything in it, read-only files too). It refuses the
  project's folder, one holding it, and drives.
- **`processes`** to `list` the running processes (`name` filters them)
  or `end` some (`name`, a program such as `rain.exe`, or `ids`); never
  buckram or what started it.
- **`edit`** for several exact replacements that must all happen or none:
  each edit is `{file, old, new}`, and its old text must be in its file
  exactly `count` times (1 by default). An empty old text makes a new
  file. An edit with another field (`path`, say), or without one of
  those three, is refused before anything is written.
- **Long output** is cut in the middle past `output_limit` bytes; the
  note at the cut names the file with all of it
  (`output_folder/<tool>.txt`). Read that file rather than running the
  program again.

A tool's result is an error when its program exits with a code other than
0 or runs past its timeout; the output is still there.

- **The call log:** every call is a line of `output_folder/calls.jsonl`
  (`{"time", "tool", "arguments", "ms", "error", "bytes"}`). Read it now and
  then for what repeats (the same `run`, a chain of calls that's one job)
  and propose a task, or a tool of buckram's own, for it.

## Setting it up in a project

1. **Nothing to build:** `bin/buckram.exe` and `bin/buckram_core.exe`
   (Windows), and `bin/buckram` and `bin/buckram_core` (Linux, x86-64),
   are in the repository (build them again only after changing `src/`;
   see below).
2. **Write the project's `buckram.json`**, in its folder, from the commands
   the project's documentation (or its scripts) give for building, testing
   and so on:
   ```json
   {
     "output_limit": 8192,
     "output_folder": "build/buckram",
     "tasks": {
       "build": { "description": "Builds it: errors and warnings, or \"built\".",
                  "run": ["tools/build.exe"] },
       "test":  { "description": "Runs the tests; given names, only those.",
                  "run": ["build/tests.exe"], "arguments": "list", "timeout": 900 }
     }
   }
   ```
   A task's `run` is the program and its arguments (a program given as a
   path is the project's); `arguments` is `"none"` or `"list"` (more
   arguments added after its own); `timeout` is in seconds (600 by
   default; a call can give its own). Say in each
   description what it gives back and what arguments it takes, since
   that's what the next agent reads. Call real programs, not `.bat` or
   `.cmd` files: those go through `cmd.exe`, which parses their command
   line again (`%`, `^`, `&`). Put `output_folder` somewhere git ignores.
3. **Register it** in the project's `.mcp.json`, with the project's
   folder as its argument (so it doesn't matter where the client starts
   it):
   ```json
   {
     "mcpServers": {
       "buckram": { "type": "stdio", "command": "c:/repos/buckram/bin/buckram.exe", "args": ["c:/path/to/project"] }
     }
   }
   ```
   On Linux, the command is `bin/buckram` (no `.exe`). Keep any servers
   already there. This is the person's configuration: ask before changing
   it.
4. **Restart the client** so it starts buckram, and check a task and
   `git status` work through it.
5. **Turn the shells off**, if the person wants (ask first: it's their
   settings). In Claude Code, in the project's
   `.claude/settings.local.json`:
   ```json
   {
     "permissions": { "deny": ["Bash", "PowerShell"] },
     "env": { "CLAUDE_CODE_USE_POWERSHELL_TOOL": "1" }
   }
   ```
   merged with what's there. Deny both, or neither: on Windows, a deny of
   Bash alone turns PowerShell off too unless that variable is set. Files
   are read and written with the client's own file tools.
6. **Tell the project's next agent**: a line in its `AGENTS.md` (or
   `CLAUDE.md`) saying its commands are buckram tasks.

## Working on buckram

- **The files:** [src/files.rain](src/files.rain) has the `files` and
  `processes` tools. [src/stub.rain](src/stub.rain) is the stub
  (`bin/buckram.exe`), which the client starts and keeps running: it
  answers `initialize`, and for every other message starts
  `bin/buckram_core.exe` afresh in the project's folder with the message as
  its standard input. [src/core.rain](src/core.rain) has the tools.
  [build.rain](build.rain) builds both into `bin/`, for Windows and for
  Linux, and those programs are committed, so buckram is used as it's
  downloaded. Code that differs by system is in `#if #OS`, or better, in
  the modules (System's `processes`, File's `make_writable`). It's written in 'rain: its
  compiler's `AGENTS.md` says how to write it, and `rain -query:module
  Json` (or `System`, `File`, `String`) lists what the modules offer.
- **Building:** `rain build.rain` once, then `build.exe [path to
  rain.exe]`, on Windows (the compiler's own system): it builds the Linux
  programs too, through `-os:linux`. Windows won't overwrite a running exe but will rename one,
  so `build.exe` moves one that's in use into `bin/old_exes` (clearing
  out those no longer running) and builds in its place. Through buckram:
  `run` with program `c:/repos/buckram/build.exe`. Commit the rebuilt exes
  in `bin/` with the change to `src/`.
- **What needs a restart:** a rebuilt core is in use from the next call,
  and so is a changed `buckram.json`, and the stub tells the client its
  tools may have changed. A change to the stub (`src/stub.rain`: its
  instructions, `initialize`, how it starts the core) needs the client to
  restart, as the running stub is the old one until then.
- **Testing:** feed a session to it, a JSON-RPC message a line, and read
  what it writes back:
  ```
  {"jsonrpc":"2.0","id":0,"method":"initialize","params":{"protocolVersion":"2025-06-18","capabilities":{}}}
  {"jsonrpc":"2.0","method":"notifications/initialized"}
  {"jsonrpc":"2.0","id":1,"method":"tools/list"}
  {"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"run","arguments":{"program":"cmd","arguments":["/c","echo","1"]}}}
  ```
  through `run` (program `c:/repos/buckram/bin/buckram.exe`, arguments the
  project folder, `input` the session). Or build a core of your own
  (`rain src/core.rain -out:scratch/core.exe`) and give it one `tools/call`
  line as `input`, with `folder` the project: it answers that one message.
  Use a scratch folder with its own `buckram.json` for tests that change
  tasks, and a scratch repository for the git tool's, so the real ones
  aren't touched. On Linux, through WSL: `run` with program `wsl.exe`,
  arguments `-e`, `/mnt/c/repos/buckram/bin/buckram` and the scratch
  folder's `/mnt/c/...` path, and `input` the session.
- **Its standard output is only protocol messages,** one a line; anything
  else goes to standard error.
- **Files are LF only** (`.gitattributes` makes sure): 'rain rejects a CR.
