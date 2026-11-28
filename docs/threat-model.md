# Threat model

This page says what hermy defends against and what it does not, and how to sort a reported gap.
[How hermy enforces its rules](enforcement.md) has what each layer does. This page is the line between a bug and a ceiling.

The boundary is the host's sandbox for shell commands and the edit guard and permission prompts for the file tools.
Inside it is anything an agent can do through its tools, honest or injected.
Outside it is anything that needs a shell write the sandbox blocks, no sandbox at all, or `bypassPermissions`.
The sandbox bounds shell commands and their child processes only. The file tools (Edit, Write) run outside it, under
the edit guard and the host's permission rules, which auto mode loosens and `bypassPermissions` switches off.
On Claude Code, `sandbox.allowUnsandboxedCommands: false` keeps a command from leaving the sandbox at all, which is
what keeps the boundary in place. Unless it is false, a command can be rerun outside the sandbox, with a prompt
unless the user has switched the prompts off.

## In model

An agent, acting through its tools inside the sandbox, or through the file tools under the guard and the host's
prompts, whether it means to or not.

Honest mistakes:

- An edit in the main checkout, outside `.worktrees/<issue>/`.
- A commit or push on the default branch.
- An edit in the worktree of an issue nobody claimed, or another session claimed.
- A weakened `.hermy.yml`: a removed file, or `guard: warn` written in.
- A shell command with a side effect it did not foresee: a new worktree, a changed `core.hooksPath`,
  a planted file a hook then trips over.

An injected agent:

- Text in an issue, a PR, a file or a web page steers the agent. It then does what a careless agent could do,
  on purpose, with the same tools and the same sandbox.
- It can plant what a sandboxed command can write: a `bash` or `jq` in the project, a FIFO at a snapshot name,
  a link where a hook reads, a `commondir` in a `.git`, a rewritten lock or settings file.
- Under auto mode, a file-tool write outside the project, which the edit guard lets through
  ([Edit guard](enforcement.md#edit-guard)) and the sandbox does not see: the user's git or jj config, say.

## Out of model

Nothing in hermy is built to stop these. A report that needs one is not a hermy bug.

- A shell write the sandbox blocks. The agent cannot make it, so no layer has to catch it.
  A file-tool write is not one of these: the sandbox does not bound it.
- `bypassPermissions`, whole. The user has switched off the host's prompts, and with them the prompt on a command
  rerun outside the sandbox, so nothing in hermy is required to hold there.
- The sandbox off or bypassed (a command that auto mode lets run outside it, or one rerun outside it unless
  `sandbox.allowUnsandboxedCommands` is false), or a host with none on. One command can then remove the config and every leftover
  ([Config lock](enforcement.md#config-lock)), and the hooks' own `PATH` and install are within reach.
- A background or hand-run process. The tripwire sees a command at its start and its end
  ([what it doesn't catch](enforcement.md#shell-tripwire)).
- The user's own terminal. Everything hermy asks you to run there, `/hermy:start` once for the git hooks
  ([Git hooks](enforcement.md#git-hooks)) or the steps of [leaving the workflow](enforcement.md#leaving-the-workflow),
  runs as you, outside any sandbox.

The sandbox differs per host. Claude Code's keeps writes out of `.git/hooks` and `.git/config`.
Copilot CLI's `--sandbox` does not, so there a sandboxed command can rewrite the git hooks
([per host](enforcement.md#per-host)). That puts the rewrite in model on Copilot CLI and out of it on Claude Code.

## What holds outside the model

Three things hold whatever the agent does inside the sandbox, which is why a gap in the local layers is not fatal.

- The sandbox, for shell commands, while they cannot leave it (`sandbox.allowUnsandboxedCommands: false` on Claude
  Code). It keeps what it blocks out of reach: on Claude Code, `.git/hooks` and `.git/config`, where the
  git hooks are named ([Git hooks](enforcement.md#git-hooks)). It is the host's, not hermy's, and it differs per host.
- Branch protection. It holds a push to the default branch that a command makes with `--no-verify`, under jj, or by
  `git send-pack` ([Branch protection](enforcement.md#branch-protection)).
- The PR check. It ties a PR to an open, assigned issue, for a cloud agent or a branch hermy did not name
  ([PR check](enforcement.md#pr-check)).

Branch protection and the PR check hold only under the settings in
[What the hard layers need](enforcement.md#what-the-hard-layers-need). Without them only the local layers hold.

## Prompt injection

In scope. An injected agent has the same tools as an honest one, so what stops the honest mistake has to stop it too,
and a layer that holds only against an agent that is not steering is a layer that does not hold.

The cost is the hardening against what a sandboxed command can plant. The hooks run outside the sandbox,
so anything such a command can write is read by code that is not sandboxed. That is why the hooks drop `PATH` entries
inside the project, refuse a FIFO before git runs, treat a link or `commondir` in a `.git` as odd, and compare the
config against a snapshot rather than trust it. Each costs code and sometimes a refusal that blocks a legitimate
toolchain ([bash and `PATH`](enforcement.md#bash-and-path)). That hardening stays.

What an injected agent does through its tools is sorted like any other gap (below): held by nothing, it is a bug.
The ceilings stay recorded, though: claims are cooperative, and the hooks are
[not a sandbox](enforcement.md#what-holds-against-what), so an agent set on getting round a cooperative rule
is a ceiling where a remote layer holds the outcome, not a bug in the hook.

## Sorting an issue

Ask three questions in order.

1. Does it need something out of model? Then there is no issue. Say which item and close it.
2. Is it in model and held by a remote layer (branch protection, the PR check)? Then it is a ceiling.
   Record it in [enforcement.md](enforcement.md), or leave it where it is already recorded. No fix.
3. Is it in model and held by nothing? Then it is a bug. Fix it, or cut the rule it sits on.

A gap already listed in enforcement.md or [enforcement-limits.md](enforcement-limits.md) is known. Link it and do not
file it again. If it is in model and no remote layer holds it, it is a bug already recorded, open until fixed or its rule
is cut, as the FIFO's remainder below. `bypassPermissions` is out of model, whole. Auto mode stays in model for what it lets
through inside the sandbox and through the file tools, every file-tool write the edit guard lets through included;
a command it lets run outside the sandbox is the sandbox bypassed, so out of model.

Worked examples, each checked against enforcement.md:

- **A FIFO at a snapshot record's name.** In model: a sandboxed command can make one. No remote layer holds it.
  `trip` opens only plain files and `snap` writes each record as a new file, so the planted name never hangs `trip`.
  The window between `snap` clearing a name and writing it holds `snap` until its 60-second timeout, which Claude Code
  blocks from 2.1.295. Before that, and on Copilot CLI, the command runs and `trip` reports its snapshot lost.
  That remainder is a bug already recorded ([per host](enforcement.md#per-host)). Do not file it again.
- **A `commondir` written into the project's `.git`.** In model: the sandbox does not protect it. It would point git
  at another repository's config. `guard`, `snap` and `trip` refuse before any git call and name the file to remove
  by hand. Held locally, so no bug. [enforcement.md](enforcement.md#per-host) describes this refusal.
- **A planted `bash` on `PATH`.** In model: a command can write `.venv/bin/bash`. The hooks drop every `PATH` entry
  inside the project before they look up `bash`, `git`, `jq` or `jj`
  ([bash and `PATH`](enforcement.md#bash-and-path)). Held locally, so no bug.
  An absolute entry outside the project that a settings `env` sets still reaches them. That is in model, and no remote
  layer holds it: a bug already recorded ([bash and `PATH`](enforcement.md#bash-and-path)), which the tripwire reports
  and the host's protected-path prompt gates for an Edit
  ([settings files](enforcement-limits.md#settings-files)). Do not file it again.
- **`--no-verify`.** In model: the agent can run it inside the sandbox. It skips the git hooks
  ([Git hooks](enforcement.md#git-hooks)). Branch protection and the PR check hold the default branch.
  A ceiling to record, which it is. No fix.
- **A command that swaps the guard's checked target for a link before the tool writes.** In model, and no remote layer
  holds it. Listed under the edit guard's "Doesn't catch" ([Edit guard](enforcement.md#edit-guard)): a bug already
  recorded. Do not file it again.
- **A shell write inside another session's worktree, or an unclaimed one.** In model, and no remote layer holds it.
  Listed under the tripwire's "Doesn't catch" ([Shell tripwire](enforcement.md#shell-tripwire)): already recorded.
- **A file-tool write of the user's git or jj config under auto mode.** In model: the edit
  guard lets files outside the project through, and auto mode loosens the prompts that would gate the write.
  Under `bypassPermissions` the same write is out of model. A Write puts
  `core.fsmonitor` into `~/.gitconfig` or `$XDG_CONFIG_HOME/git/config`. `homes()` refuses only a `HOME` or
  `XDG_CONFIG_HOME` that lies in the project, so the next `snap` runs `git status`, which runs that program outside
  the sandbox. No remote layer holds it, and enforcement.md does not record it: a bug.
- **A background command that writes later.** Out of model, as a background process. No issue
  ([Shell tripwire](enforcement.md#shell-tripwire)).
- **A command run with the sandbox off that removes `.hermy.yml` and every leftover.** Out of model. Nothing then
  tells it from a project never onboarded ([Config lock](enforcement.md#config-lock)). No issue.
