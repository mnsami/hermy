# Fallback lines run the plugin's own hermy (#157)

## Intent

On a host that does not inject or expand (Copilot, any agent reading `AGENTS.md`), the skills' fallback lines and the `AGENTS.md` block run bare `hermy <verb>`, so `PATH` picks the binary: a project's own `bin/hermy` (a `.` or `./bin` entry), a stale `get.sh` install, or none. `sweep` writes. Only the right `hermy` may run.

## Goals

- Every skill's fallback line runs `'<plugin dir>/bin/hermy' <verb>` by absolute path, single-quoted, a `'` in it written `'\''`, `<plugin dir>` being the directory that holds this skill's `skills/<name>/SKILL.md`, never one inside the project, the main checkout (the parent of `git rev-parse --path-format=absolute --git-common-dir`, spelled with `pwd -P`), nor inside the checkout the agent runs in, each compared physically, links followed. Unknown, or no `bin/hermy` there → stop and say so. A `git rev-parse` there that fails or prints anything but one absolute path (a jj repository with no `.git`, a git before 2.31) → stop and say git cannot place the project there. That definition and its stops apply only under the fallback conditions, so a host that injects never stops on it. Bare `hermy` is never run from a skill.
- The `AGENTS.md` block, read by agents that may have no plugin, runs hermy only by the file the absolute path `command -v hermy` prints reaches, resolved once and single-quoted (`'<hermy>' status`), a `'` in it written `'\''`, never a bare `hermy`, and only when that path, every link on the way and the file each lie outside the main checkout (the parent of `git rev-parse --path-format=absolute --git-common-dir`, spelled with `pwd -P`), every worktree of the repository (`git worktree list`), the checkout the agent runs in and its working directory, compared by identity (`-ef`) on the directory holding each and every one above it, so a firmlink or bind-mount spelling and an in-repository link to a file outside both stop. A relative path or one inside → stop, never run it, and tell the user to remove that `PATH` entry or put the install's directory ahead of it; a `git` command there that fails, or a `git rev-parse` printing anything but one absolute path (a jj repository with no `.git`, a git before 2.31) → stop, never run it, and tell the user git cannot place the repository there; no output, an alias, or a `readlink` or `pwd -P` that fails → stop, never run it, and tell the user to install or update hermy with `get.sh` (the plugin's repository), the user's step, never the agent's.
- A hermy skill listed for the session wins over the `AGENTS.md` block: the agent loads it and follows its fallback line, and `<hermy>` in the block is then `<plugin dir>/bin/hermy`, `<plugin dir>` as that line defines it.
- The self-test fails a fallback line that runs bare `hermy <verb>`, runs a double-quoted hermy path, or drops the plugin-dir path, and an `AGENTS.md` block without the `command -v hermy` check.

## Non-goals

- No version check: the plugin's own `bin/hermy` is the plugin's version by construction.
- No new verb or flag in `bin/hermy`.
- The Claude Code path (shims by absolute path) is unchanged.
- Rewriting `AGENTS.md` in projects already onboarded: `onboard` stops whenever a config exists, so an existing block stays as it is until the user starts over or pastes the new block.

## Approaches

- A, chosen: plugin path in the skills, `PATH` check in `AGENTS.md` alone. No version skew, no extra `doctor` run, and the plugin dir is needed by the skill anyway for its briefs, templates and `policy.md`.
- B: check, then bare `hermy`, everywhere, with `hermy doctor`'s `version:` held to the plugin's `plugin.json`. Needs the plugin dir all the same, runs `doctor` before every skill, and still stops where A would run the right binary.

## Acceptance

- Every fallback line names `'<plugin dir>/bin/hermy'` for each verb it runs, says where `<plugin dir>` is, never inside the project, unknown → stop, and carries no backticked bare `hermy <verb>`; the self-test fails without it.
- The `AGENTS.md` block carries the `command -v hermy` check (absolute, outside the main checkout, its links on the way and the file they reach), the stop and the `get.sh` remedy, and says a listed skill's fallback wins; the self-test fails without it.
- The self-test fails a fallback line with a bare `hermy status`, one without the plugin-dir path, and an `AGENTS.md` block without the `command -v hermy` check.
- CLAUDE.md's Fact injection paragraph says the same.
- verify passes.
- Stage-2 re-run (Copilot, owner-gated): with no plugin `bin/` on `PATH` and a stale install present, `next` runs the plugin's own `bin/hermy`, never the stale one.
- The `<plugin dir>` definition and its stop apply only under the fallback conditions, so a host that injects never stops on it and `claude --plugin-dir . -p "/hermy:next"` in this repository still runs.
- Each fallback line's "never inside the project" compares the plugin dir and the project physically, links followed.
- The `AGENTS.md` block runs the absolute path `command -v hermy` printed, links followed, wherever it runs hermy, never bare `hermy` after the check, and compares it against the main checkout by identity.
- The self-test fails a fallback line whose plugin-dir stop sits outside the fallback conditions, one whose `skills/<name>/SKILL.md` names another skill, and an `AGENTS.md` block that runs bare `hermy` after the check.
- The self-test holds the fallback condition, the `<plugin dir>` definition and its stop as one unbroken string, and fails a line whose stop is moved into a sentence of its own outside the conditions.
- The `AGENTS.md` block's resolved hermy also lies outside the checkout the agent runs in and outside its working directory.
- The `AGENTS.md` block's remedy splits by cause: a relative path or one inside → remove that `PATH` entry or put the install's directory ahead of it; no output → tell the user to install or update hermy with `get.sh`, the user's step, never run by the agent.
- CLAUDE.md's Fact injection paragraph says the same as the above, and this document states the shipped behaviour.
- The block gives the command for the main checkout (the parent of `git rev-parse --path-format=absolute --git-common-dir`, spelled with `pwd -P`), and each fallback line says the project is that main checkout, also held against the checkout the agent runs in.
- The block holds the path `command -v hermy` printed, every link on the way and the resolved file outside the main checkout, every worktree of the repository (`git worktree list`), the agent's checkout and its working directory, each compared by identity (`-ef`), so a firmlink or bind-mount spelling and an in-repository link to a file outside both stop.
- The block claims no stop on a stale install, which nothing checks, and gives way to any hermy skill listed for the session, not only a loaded one.
- Every command a fallback line or the block gives single-quotes the path (`'<plugin dir>/bin/hermy' config`, `'<hermy>' status`).
- The self-test fails a block missing the main-checkout command, the identity comparison, the worktrees, the links on the way, the listed-skill rule or the quoting, one claiming a stale-install stop, and a fallback line without the quoting or the project definition.
- With a hermy skill listed for the session, the block says `<hermy>` is that skill's `<plugin dir>/bin/hermy`, so no line runs an undefined `<hermy>`.
- At a plain jj root (`.jj/repo`, no `.git`), or where git cannot give one absolute path (git before 2.31 prints `--path-format=absolute` back), the block and each fallback line stop naming that cause, never with the `get.sh` remedy.
- The block's stop branches cover only their named causes, so a path that passes every check reaches its Otherwise branch.
- The self-test fails a fallback line that runs a double-quoted hermy path such as `"bin/hermy" config`.
- The block and each fallback line write a `'` inside a quoted path as `'\''`, and the self-test requires it.

## Left to the builder

The exact wording of each line, kept to the existing fallback shape and the self-test's single-line rule.
