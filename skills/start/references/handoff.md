# Hand-off

`start` reads this file at step 10.6. `$issue` here stands for the id of the issue `start` runs on: nothing fills it in for this file.

Reached from 10.5 or from a refused `round=` → list each blocking finding with its commit and ask, with no other round on offer:
- Cut: drop or narrow the rule the blocking findings sit on. Recommended when one sits on lines a harden or fix pass added, judged from what this session saw: harden's `Hardened:` lines and the fix-mode results name the files and lines those passes changed; a resume has not seen them, so there the recommendation says it is a guess. Post the whole plan, the cut made in its acceptance criteria and every other line kept (`Tier:` included), as a new `<!-- hermy:plan -->` comment first, since 10.2 reads the criteria from it, then `hermy:implement` in fix mode as 10.5 runs it, the cut as its one finding, `C1 | high | <file:line> | <the rule the cut drops or narrows> | <the cut> | <commit>`, the rule's line and its commit, or `-` for both when it sits on no one line, then one round of step 10, entered anew, whose blocking findings come back here unfixed.
- Fix by hand, then run `/hermy:start $issue` again.
- Step 11 as is: that writes `phase=done` with the blocking findings left listed in `summary` as `left blocking: <keys>`.
