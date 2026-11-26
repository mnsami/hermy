---
name: triage
compatibility: GitHub plugin for Claude Code or Copilot CLI github-mcp-server; GitLab https://<host>/api/v4/mcp; Jira https://mcp.atlassian.com/v2/mcp
description: Turn review findings into tracker issues
when_to_use: Use when the user asks to file, log or create issues or tickets from findings, a review, an audit or a list of problems.
allowed-tools: Bash(bash "${CLAUDE_PLUGIN_ROOT}/scripts/config.sh"), Bash(bash "${CLAUDE_PLUGIN_ROOT}/scripts/issue.sh" status), Bash(bash "${CLAUDE_PLUGIN_ROOT}/scripts/issue.sh" state *), mcp__plugin_github_github__search_issues, mcp__plugin_github_github__issue_write, mcp__plugin_atlassian_atlassian__getAccessibleAtlassianResources, mcp__plugin_atlassian_atlassian__atlassianUserInfo, mcp__plugin_atlassian_atlassian__searchJiraIssuesUsingJql, mcp__plugin_atlassian_atlassian__createJiraIssue, mcp__plugin_gitlab_gitlab__list_work_items, mcp__plugin_gitlab_gitlab__save_work_item
---

# Triage

## Config
!`bash "${CLAUDE_PLUGIN_ROOT}/scripts/config.sh"`

## Worktrees
!`bash "${CLAUDE_PLUGIN_ROOT}/scripts/issue.sh" status`

Only where a section above shows a `bash` command rather than its output, or `CLAUDE_PLUGIN_ROOT` is unexpanded: `<plugin dir>`, every path a `git` or `jj` command prints and the directory you run in go into every command single-quoted, each `'` in it written `'\''`; `<plugin dir>` is the directory holding this skill's `skills/triage/SKILL.md`, the hermy plugin's own install directory, never one at or inside the project (the main checkout: the parent of `git rev-parse --path-format=absolute --git-common-dir`, spelled with `pwd -P`, or, where its first line is `--path-format=absolute`, as a git before 2.31 prints it, `cd` from the directory you run in into the parent of plain `git rev-parse --git-common-dir` and take `pwd -P`), at or inside the checkout you run in (`git rev-parse --show-toplevel`) or at or inside a worktree of this repository (`git worktree list`, and `jj workspace list --ignore-working-copy --no-pager -T 'if(root, root ++ "\n")'` in a jj repository), each compared physically, links followed; unknown, or no `bin/hermy` there → stop and say so; `jj workspace root --ignore-working-copy --no-pager` succeeding and lying physically below `git rev-parse --show-toplevel` (a jj repository nested in another git repository), or either git placement above failing or giving anything but one absolute path (a jj repository with no `.git`) → place them through jj: the main checkout is `jj workspace root --ignore-working-copy --no-pager` run in the default workspace, or the directory `.worktrees/` sits in, the checkout you run in `jj workspace root --ignore-working-copy --no-pager`, and the worktrees `jj workspace list --ignore-working-copy --no-pager -T 'if(root, root ++ "\n")'` alone, never `git worktree list`; where `git rev-parse --show-toplevel` holds a `.hermy.yml` (a git project holding a nested jj repository), git's main checkout and that top level count among them too; git's placement only where jj gives no root or one at or above git's; neither git nor jj placing them → stop and say neither can place the project here. A call to `HERMY_AGENT_DIR=.claude '<plugin dir>/bin/hermy' <verb>` denied as a path outside the working directory (Copilot's `--allow-all-tools` does not cover one) → stop and say this session needs `--add-dir '<plugin dir>'`. A section above shows a `bash` command rather than its output → this host does not inject: run `HERMY_AGENT_DIR=.claude '<plugin dir>/bin/hermy' config` and `HERMY_AGENT_DIR=.claude '<plugin dir>/bin/hermy' status` and read their output as Config and Worktrees. `CLAUDE_PLUGIN_ROOT` unexpanded → `<plugin dir>`, never a `bin/hermy` or `skills/` found inside the project; in a command run `HERMY_AGENT_DIR=.claude '<plugin dir>/bin/hermy' <verb>` in its place (`issue.sh <verb>` → `HERMY_AGENT_DIR=.claude '<plugin dir>/bin/hermy' <verb>`, `config.sh` → `HERMY_AGENT_DIR=.claude '<plugin dir>/bin/hermy' config`). `$ARGUMENTS` unexpanded → take it from the user's request.

`NOT_ONBOARDED` → stop and say: run `/hermy:onboard` first.
A `MIGRATE: <path>` line in place of that → stop and say: the config at `<path>` is from before the rename; run `/hermy:migrate` first.
A `git_dir:` line in place of that → stop and show that line as it is.
`tracker: jira` → every Atlassian MCP call takes `cloudId`: call `getAccessibleAtlassianResources` once and use the entry whose url is `jira.site`; me is `atlassianUserInfo`'s `accountId`. Those tools not loaded → stop and say the atlassian plugin needs to be authenticated.
`tracker: gitlab` → every GitLab MCP call takes `project_id`, which is `repo`. Those tools not loaded → stop and say the gitlab plugin must be enabled and signed in. In a `description` sent to `save_work_item`, a line starting with `/` goes in backticks: it rejects such a line.

## Source
Findings come from this conversation, or from `$ARGUMENTS` when given (a file path or pasted text), and from the `triage` lines of `bash "${CLAUDE_PLUGIN_ROOT}/scripts/issue.sh" state <issue>` for each claimed issue of mine in Worktrees (`owners` is `me`; empty `triage` lines are ignored): say which issue they came from.

## Steps

1. List the findings, one issue per finding. Merge duplicates. A finding that is another case of a limit the project's docs already record (a known ceiling, a limits page) → no issue; say where it is recorded.
2. Search for each finding, never one list of all open issues (it stops at a page and misses the rest): 2-4 distinctive words from the finding, then github `search_issues` with `owner` and `repo` from `repo`, `query: "is:open in:title <words>"` and `fields: [number, title, labels]`; jira `searchJiraIssuesUsingJql` with JQL `project = <project> AND statusCategory != Done AND summary ~ "<words>"`; gitlab `list_work_items` with `state: opened`, `first: 100` and `search: "<words>"`. No hit → once more with other words. A finding a hit already covers is skipped and cited.
3. Show a table: title, label, covered-by. Ask once to confirm.
4. Create each issue (github `issue_write` `create` with `title`, `body` and `labels`; jira `createJiraIssue`; gitlab `save_work_item` with `title`, `type_name: Issue`, `description` and `labels`) with this body:
   - **Problem**: what is wrong, with evidence (`file:line`, command output).
   - **Fix**: checklist of concrete steps.
   - **Acceptance**: checklist of observable checks.
   Labels: `bug` or `enhancement` (github, gitlab); issue type Bug or Task (jira). gitlab: `save_work_item` refuses a label the project does not have → create it without `labels` and say which was left off.
5. Each issue whose `triage` lines were used: lines that were filed, skipped as already covered or skipped as a limit the docs already record are removed; any others (dropped at the confirm, or a failed create) stay. One call: `bash "${CLAUDE_PLUGIN_ROOT}/scripts/issue.sh" state <issue> triage=` followed by `"triage+=<line>"` for each remaining line.
6. Print the created links.
