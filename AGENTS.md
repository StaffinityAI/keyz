# AGENTS.md

Instructions for coding agents working in this repository.

## Project

keyz — macOS key remapping daemon in Zig (0.17.1), a reimplementation of
[keyd](https://github.com/rvaiya/keyd). Config format must stay compatible with
keyd v2; see README.md for goals and the keyd man page for the format spec.

- Library module: `src/root.zig` (exported as `keyz`)
- Executable: `src/main.zig`

## Commands

```sh
zig build            # build into zig-out/
zig build run -- ... # run the executable
zig build test       # run all tests (library + executable)
```

Run `zig build test` before considering a change done.

## Issue tracking

Work is tracked with git-native issues (`refs/issues/*`), not GitHub Issues.

- Use the `git-issue` MCP tools when available; otherwise the `git issue` CLI
  (`git issue ls`, `show`, `create`, `comment`, `edit`, `close`, `tree`,
  `milestone`).
- Before starting work, find or create the issue for it. Reference the short
  id in commit messages (e.g. `Implement parser (#a1b2c3d)`).
- Large work: one parent issue per milestone/epic, child issues per task. Use
  `git issue tree` to view progress and blockers.
- Record decisions and findings as issue comments, not only in chat.
- Close an issue only when its work is merged/committed and tests pass.
- Sync with the remote via `git issue sync origin` (push issues together with
  code).
