# Fallback lines run the plugin's own hermy (#157)

## Intent

On a host that does not inject or expand (Copilot, any agent reading `AGENTS.md`), the skills' fallback lines and the `AGENTS.md` block ran bare `hermy <verb>`, so `PATH` picked the binary: a project's own `bin/hermy` (a `.` or `./bin` entry), a stale `get.sh` install, or none. `sweep` writes. Only the right `hermy` may run.

## Shipped behaviour

- Every skill's fallback line runs `'<plugin dir>/bin/hermy' <verb>` by absolute path, single-quoted, a `'` in it written `'\''`, `<plugin dir>` being the directory that holds this skill's `skills/<name>/SKILL.md`, never one inside the project, the main checkout (the parent of `git rev-parse --path-format=absolute --git-common-dir`, spelled with `pwd -P`), nor inside the checkout the agent runs in, each compared physically, links followed. Unknown, or no `bin/hermy` there → stop and say so. A `git rev-parse` there that fails or prints anything but one absolute path (a jj repository with no `.git`, a git before 2.31) → stop and say git cannot place the project there. That definition and its stops apply only under the fallback conditions (a section shows its command rather than its output, or `CLAUDE_PLUGIN_ROOT` is unexpanded), so a host that injects never stops on it. No fallback line runs a bare `hermy`.
- The `AGENTS.md` block never runs `hermy` from `PATH` and never resolves it there (`command -v hermy`). The hermy plugin's own `config` skill listed for the session is the agent's source, read for its injected Current and Worktrees sections alone, whose commands only read, never for its steps, where the host injects, so an agent that injects (Claude Code reading the block) runs nothing undefined, else its fallback line's `'<plugin dir>/bin/hermy'`. One skill, named: any hermy skill would let the agent load `next`, `start` or `refresh`, whose injected `sweep` removes merged worktrees, or `ship` and `drop`, which run only on the user's ask. Never a directory-scoped copy (listed with a path prefix) nor one whose `SKILL.md` lies inside the main checkout, the checkout the agent runs in or a worktree of the repository, compared physically, links followed: a sandboxed command or a merged PR can plant one (`.claude/skills/`, `.github/skills/`) whose fallback line runs the project's own `bin/hermy`. Without such a skill, `get.sh`'s install. Otherwise the block runs `get.sh`'s install, the path `${HERMY_HOME:-$HOME/.hermy}/bin/hermy` expands to, single-quoted (`'<hermy>' config`), a `'` in it written `'\''`; missing or not executable → stop and tell the user to install or update it with `get.sh` from https://github.com/mnsami/hermy, the user's step, never the agent's. Its "Not installed" line, for a missing plugin, says the same.

## Non-goals

- No version check: the plugin's own `bin/hermy` is the plugin's version by construction, and `get.sh`'s install is the one the user chose.
- No new verb or flag in `bin/hermy`.
- The Claude Code path (shims by absolute path) is unchanged.
- Rewriting `AGENTS.md` in projects already onboarded: `onboard` stops whenever a config exists, so an existing block, the `command -v` one or the bare-`hermy` one before it, stays as it is until the user starts over or pastes the new block.

## Approaches

- A, chosen: the skills call `<plugin dir>/bin/hermy` by absolute path; the `AGENTS.md` block runs `get.sh`'s fixed install or defers to a listed skill, with no `PATH` lookup.
- B, cut: the `AGENTS.md` block resolves `command -v hermy` and walks every link and directory on the way against the main checkout, every worktree, the agent's checkout and its working directory by identity, as the core does for `HERMY_EXEC`. Five review cycles kept finding cases the prose walk missed (a directory link on `PATH` into the project pointing outside; `<hermy>` undefined on an injecting host that reads `AGENTS.md`), while a fixed path has nothing to walk.

## Ceilings

- `HERMY_HOME` is trusted as `get.sh` trusts it: one set by the project's own environment (a `.envrc`, an agent settings `env`) points the block at whatever it names, a path inside the project included.
- An agent with no hermy skill listed and no `get.sh` install stops; it never falls back to `PATH`.

## Acceptance

- Every skill's fallback line names `'<plugin dir>/bin/hermy' <verb>` for each verb it runs, says where `<plugin dir>` is, never inside the project, unknown → stop, and holds no bare `hermy <verb>`, quoted or not.
- The `<plugin dir>` definition and its stops apply only under the fallback conditions, so a host that injects never stops on it and `claude --plugin-dir . -p "/hermy:next"` in this repository still runs.
- Each fallback line defines the project as the main checkout, also holds the plugin dir outside the checkout the agent runs in, compares physically with links followed, and stops naming the cause where git cannot place the project (a plain jj root, a git before 2.31).
- Every command a fallback line or the block gives single-quotes the path, a `'` in it written `'\''`.
- The `AGENTS.md` block never runs `hermy` from `PATH` and never uses `command -v hermy`.
- With the hermy plugin's own `config` skill listed for the session, the block defers to it: its injected Current and Worktrees sections alone, never its steps, where the host injects, else its fallback line's `'<plugin dir>/bin/hermy'`; never a directory-scoped copy nor one inside the main checkout, the agent's checkout or a worktree.
- Otherwise the block runs hermy as `${HERMY_HOME:-$HOME/.hermy}/bin/hermy`, expanded and single-quoted; missing or not executable → stop and tell the user to install or update it with `get.sh` from https://github.com/mnsami/hermy, the user's step, never the agent's, and its "Not installed" line says the same.
- The self-test fails a fallback line with a bare or otherwise-pathed hermy (`` `hermy ``, `bin/hermy `, `"bin/hermy"`, `./hermy`, `exec hermy`), one without the plugin-dir definition, its conditions, its stops, the project definition or the quoting, one whose `skills/<name>/SKILL.md` names another skill, and one whose stop sits outside the conditions.
- The self-test fails an `AGENTS.md` block that holds `command -v hermy` or a bare `hermy <verb>`, lacks the listed-skill rule (the plugin's own `config` skill, its sections alone, no planted copy), the `${HERMY_HOME:-$HOME/.hermy}/bin/hermy` path, the quoting or the user's-step `get.sh` remedy, or still says "get it from that repository".
- CLAUDE.md's Fact injection paragraph says the same.
- verify passes.
- Owner-run on Copilot: with no plugin `bin/` on `PATH` and a stale install present, `next` runs the plugin's own `bin/hermy`, never the stale one.

## Left to the builder

The exact wording of each line, kept to the existing fallback shape and the self-test's single-line rule.
