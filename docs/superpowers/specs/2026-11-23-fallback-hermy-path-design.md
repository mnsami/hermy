# Fallback lines run the plugin's own hermy (#157)

## Intent

On a host that does not inject or expand (Copilot, any agent reading `AGENTS.md`), the skills' fallback lines and the `AGENTS.md` block run bare `hermy <verb>`, so `PATH` picks the binary: a project's own `bin/hermy` (a `.` or `./bin` entry), a stale `get.sh` install, or none. `sweep` writes. Only the right `hermy` may run.

## Goals

- Every skill's fallback line runs `<plugin dir>/bin/hermy <verb>` by absolute path, `<plugin dir>` being the directory that holds this skill's `skills/<name>/SKILL.md`, never one inside the project, the two compared physically, links followed. Unknown, or no `bin/hermy` there → stop and say so. That definition and its stop apply only under the fallback conditions, so a host that injects never stops on it. Bare `hermy` is never run from a skill.
- The `AGENTS.md` block, read by agents that may have no plugin, runs hermy only by the absolute path `command -v hermy` prints, links followed, resolved once, never a bare `hermy`, and only when that path lies outside the repository's root (the main checkout), the checkout the agent runs in and its working directory, each spelled physically, as `HERMY_EXEC` is held. A relative path or one inside → stop, never run it, and tell the user to remove that `PATH` entry or put the install's directory ahead of it; anything else (no output, an alias) or a stale install → stop, never run it, and tell the user to install or update hermy with `get.sh` (the plugin's repository), the user's step, never the agent's.
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

- Every fallback line names `<plugin dir>/bin/hermy` for each verb it runs, says where `<plugin dir>` is, never inside the project, unknown → stop, and carries no backticked bare `hermy <verb>`; the self-test fails without it.
- The `AGENTS.md` block carries the `command -v hermy` check (absolute, outside the repository, links followed), the stop and the `get.sh` remedy, and says a loaded skill's fallback wins; the self-test fails without it.
- The self-test fails a fallback line with a bare `hermy status`, one without the plugin-dir path, and an `AGENTS.md` block without the `command -v hermy` check.
- CLAUDE.md's Fact injection paragraph says the same.
- verify passes.
- Stage-2 re-run (Copilot, owner-gated): with no plugin `bin/` on `PATH` and a stale install present, `next` runs the plugin's own `bin/hermy`, never the stale one.
- The `<plugin dir>` definition and its stop apply only under the fallback conditions, so a host that injects never stops on it and `claude --plugin-dir . -p "/hermy:next"` in this repository still runs.
- Each fallback line's "never inside the project" compares the plugin dir and the project physically, links followed.
- The `AGENTS.md` block runs the absolute path `command -v hermy` printed, links followed, wherever it runs hermy, never bare `hermy` after the check, and compares it against the repository root spelled physically.
- The self-test fails a fallback line whose plugin-dir stop sits outside the fallback conditions, one whose `skills/<name>/SKILL.md` names another skill, and an `AGENTS.md` block that runs bare `hermy` after the check.
- The self-test holds the fallback condition, the `<plugin dir>` definition and its stop as one unbroken string, and fails a line whose stop is moved into a sentence of its own outside the conditions.
- The `AGENTS.md` block's resolved hermy also lies outside the checkout the agent runs in and outside its working directory, both physically, as `HERMY_EXEC` is held.
- The `AGENTS.md` block's remedy splits by cause: a relative path or one inside → remove that `PATH` entry or put the install's directory ahead of it; no output or a stale install → tell the user to install or update hermy with `get.sh`, the user's step, never run by the agent.
- CLAUDE.md's Fact injection paragraph says the same as the above, and this document states the shipped behaviour.

## Left to the builder

The exact wording of each line, kept to the existing fallback shape and the self-test's single-line rule.
