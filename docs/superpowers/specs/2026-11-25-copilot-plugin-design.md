# hermy as a Copilot CLI plugin (#33)

Status: design, 2026-11-25. Builds on the hook contract (#278, `2026-11-11-hook-contract-design.md`) and the spike (`2026-11-11-copilot-spike-design.md`, results on #33).

## Intent

Under Copilot CLI every hermy hook is inert today: Copilot runs the exec form's `command` (`/usr/bin/env`) without its `args`, so no guard, tripwire or config lock evaluates anything and nothing tells the user. The goal is that the same checkout, installed in Copilot CLI, enforces what it enforces in Claude Code, and that every layer Copilot cannot hold is written down as a ceiling, never hidden.

## Goals

- Copilot CLI loads a hook manifest of its own that runs the same launcher, shim and core as Claude Code, with no shell between the host and the launcher's `env -u`.
- The stage 2 guardrail rows hold in a Copilot headless run (below).
- Every Copilot-only ceiling is in CLAUDE.md's Known ceilings, README and a per-host table.
- Self-test fixtures carry Copilot's own payloads through the real launcher and shim.

## Non-goals

- `doctor`'s per-agent `agent:` line (#239). #33 adds the measured Copilot facts to #239 instead.
- `tiers` for Copilot and per-spawn model names (#416).
- Whether `COPILOT_AGENT_SESSION_ID` equals the payload's `session_id`, and a Bash-tool side `HERMY_SESSION` on Copilot (#414). Until then a lock taken on Copilot names no session, so the guard does not hold edits to its owner: the no-session rows of Lock states.
- The `hooks: off` line on Copilot (`HERMY_HOOKS`): Copilot skills call `bin/hermy` directly, through no shim.
- GitLab and Jira live runs; Codex and other hosts.

## Anti-goals

- Any change to how Claude Code behaves: every existing payload test passes unchanged.
- A hook registration that goes through a shell a settings `env` can reach.
- Copilot-specific code in `bin/hermy` (#278).

## Measured on Copilot CLI 1.0.114 (2026-11-25, headless, scratch probe plugin)

- A manifest at `.github/plugin/plugin.json` is read before `.claude-plugin/plugin.json` (Copilot's documented legacy order: `.plugin/plugin.json`, `plugin.json`, `.github/plugin/plugin.json`, `.claude-plugin/plugin.json`), and its `"hooks": "hooks/copilot.json"` loads that file; `hooks/hooks.json` is then not used. Claude Code reads neither.
- A hook entry `{"type":"command","exec":"/usr/bin/env","args":[…]}` runs with no shell, and `${CLAUDE_PLUGIN_ROOT}` is expanded inside `args`; `${COPILOT_PLUGIN_ROOT}` is not.
- PascalCase events with Claude matchers fire on Claude tool names: `PreToolUse` `Edit|Write|NotebookEdit` on a create (`tool_name: Write`, `tool_input.path`, `file_text`), `Bash` on a shell command. Top-level `tool_use_id`, `session_id`, `cwd`.
- A failing shell command (`false`) fires `PostToolUse` alone, `tool_result.result_type: "success"`, the exit code only in `text_result_for_llm`. No `PostToolUseFailure`.
- `SessionStart` `source` is `new`; the hook's working directory is the plugin dir; `CLAUDE_PROJECT_DIR` is the project.
- Documented (hooks configuration reference): a timeout fails open on every event; exit 2 and any other non-zero exit of `preToolUse` deny; `timeoutSec` (default 30), `timeout` its alias; `ask` becomes a deny where no one can answer.

Measured on 1.0.115 (Copilot updated itself mid-session; the rows above hold there too):

- No plugin hook runs in a folder Copilot does not trust: the debug log shows `has_hooks:false`, `Plugin maintenance skipped {"reason":"untrusted"}`, and every guardrail is silently off. Headless runs need `--trust-folder`; interactive Copilot asks for folder trust at start.
- `PreToolUse` exit 2 blocks, and the model sees only `Denied by preToolUse hook: hook exited with code 2`: neither the stderr message nor a `permissionDecision: deny` JSON on stdout reaches it while the exit is 2.
- `PostToolUse` exit 2: its stderr shows in the user's terminal (`! <message>`), not to the model. A `{"decision":"block","reason":"…"}` on stdout is read with exit 2 as with exit 0: the tool result the model sees becomes `Tool result blocked: <reason>`. A nested `additionalContext` from `PostToolUse` does not reach the model.
- Hooks fire for a sub-agent's tool calls (a sub-agent's file create ran the `Edit|Write|NotebookEdit` hook).

## Approaches

- **A. One shim for both hosts (chosen by the owner).** `scripts/issue.sh`'s `hermy_in` also reads `tool_input.path` and turns a string `tool_input` (an `apply_patch`) into its paths. `hooks/copilot.json` runs exactly the argv of `hooks/hooks.json`, so the drift check is strict equality and the launcher is untouched. Claude's mapping also accepts Copilot's keys, which only ever adds paths to guard.
- B. A separate `scripts/copilot.sh`, picked by one extra launcher argv word; parity is equality minus that word. Two mappings to keep in step for no behaviour gained.
- Rejected before the choice: the issue's single-string `command` form. It needs a shell between Copilot and `env -u`, which the exec form exists to avoid; Copilot's `exec` + `args` makes it unnecessary.

## Decided by the owner

- Approach A.
- The builder (Claude) runs the live Copilot headless checks on the owner's account.
- `doctor`'s Copilot report is deferred to #239.
- No `SessionStart`/`link` on Copilot: `link`'s check knows Claude Code's cache layout alone, Copilot's data dir is a hash of the source path and cannot be derived from the install path, and Copilot skills call `bin/hermy start` directly, so `HERMY_EXEC` is never set there and the git hooks run the install's own `bin/hermy` path either way. Documented as a ceiling; the self-test asserts the Copilot manifest has no `SessionStart`.
- The agent dir on the skills' side is fixed in #33: every fallback line runs `'<plugin dir>/bin/hermy'` with `HERMY_AGENT_DIR=.claude` in front, as the Claude shim sets it, so `onboard`'s `exclude answers` and `forget answers`, `migrate`'s `exclude answers` and `migrate records` work on a host with no shim (they stop with `needs HERMY_AGENT_DIR` today, since #157).

## Design

### Manifest

- `.github/plugin/plugin.json`: `name` `hermy`, `description`, `hooks: "hooks/copilot.json"`. `version` and the other fields as `.claude-plugin/plugin.json` has them when Copilot needs them; the self-test asserts every field both carry is equal, so a release bump that misses one fails. RELEASING.md names both files.
- `hooks/copilot.json`: `"version": 1`, PascalCase events:
  - `PreToolUse` `Edit|Write|NotebookEdit` → `guard`, `timeout: 10`
  - `PreToolUse` `Bash` → `snap`, `timeout: 60`
  - `PostToolUse` `Bash` → `trip`, `timeout: 60` (Copilot's default is 30 s; `trip` runs the fingerprint `snap` does, on the same 20 s budget)
  - Each entry `"type": "command"`, `"exec": "/usr/bin/env"`, `"args"` identical to the `hooks/hooks.json` entry for the same verb.
  - No `PostToolUseFailure`: Copilot fires `PostToolUse` for a failing shell command, and a tool failure result would fire both, running `trip` twice. No `SessionStart`. No `onFailure` (Copilot does not read it; a non-zero exit already denies).
- Skills and agents load from the default `skills/` and `agents/` under the new manifest; if a live run shows they do not, the manifest names them.

### Shim (`scripts/issue.sh`)

- `hermy_in`: `paths` takes `.tool_input.file_path`, `.notebook_path` and `.path` from an object, and, from a string `tool_input`, every `*** Update File:`, `*** Add File:`, `*** Delete File:` and `*** Move to:` path (the extractor the self-test's `patchshim` uses today, CR trimmed). A string with no such line gives an empty list, which the guard refuses. The non-string path rule stays.
- Everything else as today, and it holds on Copilot as it stands: `CLAUDE_PROJECT_DIR` is set in Copilot's hook env; `CLAUDE_CODE_SESSION_ID` and `CLAUDE_PID` are unset there, so `HERMY_SESSION`/`HERMY_PID` are empty; `session` comes from the payload; `hermy_out`'s nested `hookSpecificOutput` envelope is honoured by Copilot (spike H7).
- `HERMY_AGENT_DIR=.claude` on both hosts. Copilot reads hooks from `.claude/settings.json` and `settings.local.json` as cross-tool config, so the fingerprint should watch them there too, and onboarding's records stay at one path. Trade-off: Copilot has no protected-path prompt for `.claude/`, so a Copilot Edit-tool write there passes the guard unprompted, the same as Claude Code's `bypassPermissions` and auto mode; documented as a ceiling.
- `HERMY_ASK` stays unset: interactive Copilot asks (without the reason text, spike follow-up), headless Copilot turns the ask into a deny by itself.

### `trip` and failure

No `failed` key. `trip` reads no exit status and runs once per command on both hosts (Claude: exactly one of `PostToolUse`/`PostToolUseFailure`; Copilot: `PostToolUse`). The hook-contract design's non-goal and `docs/specs/2026-10-15-cli-core-design.md`'s host table record it as the decision.

### `trip`'s report reaches the model on Copilot

On Copilot a `PostToolUse` exit 2 shows its stderr to the user alone, so the tripwire's and config lock's reports (what the command changed, what was put back, what to undo) never reach the model. The fix, host-neutral, no host detection:

- The neutral contract gains a fifth verdict for `trip` alone: `{"hermy":"block","text":"…"}` on stdout, printed with the same text as the stderr message, then exit 2 as today. `guard` and `snap` print none: Copilot reads no stdout on a `PreToolUse` exit 2, and turning their block into an exit 0 deny would fail open whenever the JSON is not read.
- `hermy_out` maps it to `{"decision":"block","reason":<text>}`. Copilot reads it with exit 2 (measured); Claude Code reads stderr on exit 2 and ignores stdout, so Claude behaves as today. A `claude -p` probe confirms that before it ships; if Claude reads it, it carries the same reason, so nothing changes for the model there either.
- Copilot replaces the command's output with `Tool result blocked: <text>`: the model loses that output on a tripped command. Documented.

### Guard and `snap` refusals on Copilot

Their reason never reaches the model on Copilot (`hook exited with code 2` alone). Documented as a ceiling; the `AGENTS.md` block, which Copilot loads, already names `/hermy:start` as the way in, and the user's terminal is where to look.

### Skills

The fallback lines (#157, #279) are the primary path on Copilot already. Each gains:

- `HERMY_AGENT_DIR=.claude` in front of `'<plugin dir>/bin/hermy'` (Decided by the owner). The self-test holds the prefix on every fallback line and still fails any other spelling of `hermy`.
- One clause: a call to `'<plugin dir>/bin/hermy'` denied as a path outside the working directory (Copilot's `--allow-all-tools` does not cover one) → stop and say the session needs `--add-dir '<plugin dir>'`. The self-test asserts it on every fallback line.

### Docs

- README, Copilot CLI section: install (`copilot plugin install`, and the marketplace route as measured), folder trust (interactive trust, `--trust-folder` headless: without it no hook runs), `--add-dir '<plugin dir>'`, `--allow-tool='<server>(<tool>)'` for headless MCP calls (interactive pre-approval through `allowed-tools` works, per the spike follow-up), and a link to the per-host table.
- `docs/enforcement.md`: a per-host table of each layer (edit guard, config ask, tripwire, config lock, git hooks, PR check, branch protection): blocks, asks, warns or inert, on Claude Code and Copilot CLI, plus the Copilot timeout figures measured below.
- CLAUDE.md: the deliverable list and the core/adapter paragraph name the Copilot manifest; Known ceilings gain the Copilot ones: a hook timeout fails open (the guard's 8 s and the fingerprint's 20 s budgets keep both well inside 10 s and 60 s; a repository past the fingerprint's budget runs unchecked, reported), hooks run inside Copilot's `--sandbox` where the shell can write `.git/hooks` and `.git/config`, no hook at all in an untrusted folder, a guard or `snap` refusal shown without its reason, a tripped command's output replaced by the report, no `link`, `.claude/` unprompted, and the `ask` shown without its reason.
- `docs/specs/2026-10-15-cli-core-design.md`: the Copilot row updated to the measured registration and failure-event facts.
- #239 gets a comment with the measured Copilot facts for its `agent:` line.

### Self-test

- `selftest.sh`: `hooks/copilot.json` asserted as a table like `hooks/hooks.json` (event, matcher, type, `exec`, args, timeout), each verb's `args` equal to `hooks/hooks.json`'s, no `SessionStart`, no `PostToolUseFailure`; `.github/plugin/plugin.json`'s `hooks` names it and its shared fields equal `.claude-plugin/plugin.json`'s.
- `trip`'s block verdict: a tripped command prints exactly one `{"hermy":"block",...}` line, its text equal to the stderr message, and exit 2; through the shim, one `{"decision":"block","reason":...}` line. `selftest-core.sh` still fails on a host envelope key in `bin/hermy`.
- Copilot payloads through the real launcher and shim, beside the existing Claude ones: a `Write` with `path`/`file_text` outside any worktree (refused) and in the claimed worktree (passes), an `Edit` with `path`/`old_str`/`new_str`, an `apply_patch` string (`tool_name: Edit`) touching `README.md` (refused) and the claimed worktree (passes), the config (asks), and a failing shell command's `PostToolUse` (`result_type: "success"`, `exit code 1` in its text) after a main-checkout write, which `trip` reports.
- `selftest-core.sh`'s test-only `patchshim` keeps its Codex case; its Copilot case moves to the real shim.

### Live verification (builder runs it, headless, the owner's account)

In a throwaway repo outside the checkout, onboarded with `worktree: git`, `copilot --plugin-dir <worktree> --trust-folder -p … --allow-all-tools --no-ask-user --add-dir <worktree>`:

- 3a2: edit `README.md` in the main checkout, `AGENTS.md` moved aside → refused.
- 3b: create `.worktrees/99/x.txt`, no claim → refused.
- 3c: edit `.hermy.yml` → denied (ask with no user).
- 3d: shell `echo x >> README.md` → `trip` reports it, and the model quotes the report.
- 3e: shell `rm .hermy.yml` → restored.
- 3g: two parallel shell calls → each `trip` pairs with its `snap` by `tool_use_id`, no lost snapshot.
- An `Edit` (old/new string) and an `apply_patch` (a codex model) → guard fires.
- Timeouts: a probe hook with `timeout: 60` sleeping past 30 s, then exit 2, blocks (so 60 is honoured, not the 30 s default), and one sleeping past its timeout lets the call through (fail open). The guard's and `snap`'s own worst case on a large scratch repo is measured and recorded.
- A skill (`/hermy:next` or `/hermy:config`) runs through its fallback line with `--add-dir`.

Results go in this document and the PR. The probe and script stay outside the repository, as the spike's did.

## Calls made by the designer (check these)

- `HERMY_AGENT_DIR=.claude` on Copilot's hook side: Copilot reads `.claude/settings*.json` hooks too, so the fingerprint watches them; the cost is Copilot's Edit tool writing `.claude/` unprompted.
- No `HERMY_ASK=0`: Copilot asks interactively and denies headless by itself.
- `trip` registered on `PostToolUse` alone in Copilot's manifest.
- `trip`'s new `block` verdict; `guard`/`snap` stay exit 2 without one (fail-closed beats a visible reason).
- The live probe and script stay out of the repository.

## Left to the builder

- Exact wording of docs and messages; where in README the Copilot section sits.
- Whether `version` and other fields are needed in `.github/plugin/plugin.json` (a live run decides; the equality assertion covers whichever are there).
- The marketplace route's exact commands, from a run in an isolated Copilot config if possible; otherwise documented as unmeasured.
