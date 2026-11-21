# buckram's changes

What's new in each version, newest first. The version is `VERSION`
(semantic versioning), built into buckram: MCP's `serverInfo` gives it.

Note each change under Unreleased as it's made, one line each. Installing
(`tools/install.rain`) makes them the next version's (a patch, or with
`--minor` or `--major` that part), bumping `VERSION` before the release
builds are made, and they're its Perforce changelist's description:
`buckram` and the version, then this list. Installing again before that
changelist is submitted keeps its version, and adds any new notes to it.

## Unreleased

- A task's arguments can be `"joined"`: the first joined to its run's last piece (`-test:` given `String` runs `-test:String`).
- git: a commit whose arguments give its message (`--amend --no-edit`, `-C <commit>`, `--fixup=<commit>`, `-F <file>`) needs no `message`.
- A timeout of one second says "1 second", in the result, the log and the task descriptions.

## v0.4.0

- Versions are recorded here from this one on: what came before is in git's log.
- Tasks take a `folder` to run in (a worktree's, say), where the programs they give as paths are that folder's.
- The call log says why a call failed (`"why"`), and `tools/calls.rain` sums it up.
- `files` copy or move onto a running program moves it aside into `old_exes` first.
