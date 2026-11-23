# Fallback lines run the plugin's own hermy (#157)

## Intent

On a host that does not inject or expand (Copilot, any agent reading `AGENTS.md`), the skills' fallback lines and the `AGENTS.md` block run bare `hermy <verb>`, so `PATH` picks the binary: a project's own `bin/hermy` (a `.` or `./bin` entry), a stale `get.sh` install, or none. `sweep` writes. Only the right `hermy` may run.

## Goals

- Every skill's fallback line runs `<plugin dir>/bin/hermy <verb>` by absolute path, `<plugin dir>` being the directory that holds this skill's `skills/<name>/SKILL.md`, never one inside the project. Unknown → stop and say so. Bare `hermy` is never run from a skill.
- The `AGENTS.md` block, read by agents that may have no plugin, runs bare `hermy` only after `command -v hermy` prints an absolute path that lies outside this repository with links followed; no output, a relative path or one inside → stop, never run it, and install or update through `get.sh` (the plugin's repository).
- A loaded hermy skill's own fallback line wins over the `AGENTS.md` block.
- The self-test fails a fallback line that runs bare `hermy <verb>` or drops the plugin-dir path, and an `AGENTS.md` block without the `command -v hermy` check.

## Non-goals

- No version check: the plugin's own `bin/hermy` is the plugin's version by construction.
- No new verb or flag in `bin/hermy`.
- The Claude Code path (shims by absolute path) is unchanged.
- Rewriting `AGENTS.md` in projects already onboarded: `onboard` replaces the block on its next run.

## Approaches

- A, chosen: plugin path in the skills, `PATH` check in `AGENTS.md` alone. No version skew, no extra `doctor` run, and the plugin dir is needed by the skill anyway for its briefs, templates and `policy.md`.
- B: check, then bare `hermy`, everywhere, with `hermy doctor`'s `version:` held to the plugin's `plugin.json`. Needs the plugin dir all the same, runs `doctor` before every skill, and still stops where A would run the right binary.

## Acceptance

- Every fallback line names `<plugin dir>/bin/hermy` for each verb it runs and carries no bare `hermy <verb>`; the self-test fails without it.
- The `AGENTS.md` block carries the `command -v hermy` check and the `get.sh` remedy; the self-test fails without it.
- CLAUDE.md's Fact injection paragraph says the same.
- Stage-2 re-run (Copilot, owner-gated): with no plugin `bin/` on `PATH` and a stale install present, `next` runs the plugin's own `bin/hermy`, never the stale one.

## Left to the builder

The exact wording of each line, kept to the existing fallback shape and the self-test's single-line rule.
