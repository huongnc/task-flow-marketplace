# task-flow

Claude Code plugin layered on [superpowers](https://github.com/obra/superpowers). Superpowers does planning, subagent-per-task implementation and code review. task-flow adds what it lacks.

```
1 plan (superpowers) -> 2 impact-reviewer* -> 3 implement + review (superpowers) -> 4 verifier* -> 5 user-tester* -> 07-summary.md
```
`*` = provided by this plugin.

Gates: impact REJECT returns to planning; verify/usertest failures return to implement and rerun everything after. Max 3 loops per gate, then it stops and reports. Artifacts: `.claude/runs/<slug>/` (add `.claude/runs/` to .gitignore).

Stage 5: FE tasks are tested by the agent in Chrome. BE tasks: the orchestrator asks which target (local/test/uat) and for approval first. Prod is never allowed.

## Install

```
/plugin marketplace add <owner>/<repo>     # or a local path
/plugin install task-flow@task-flow-marketplace
/reload-plugins
```
superpowers is declared as a dependency (`superpowers@claude-plugins-official`) and installs automatically. If it does not: `/plugin install superpowers@claude-plugins-official`.

Try without installing: `claude --plugin-dir ./plugins/task-flow` (superpowers must be installed).

## Use

```
/task-flow:run <task description>
```
Optional per-repo config: copy `plugins/task-flow/examples/task-flow.md.example` to `<repo>/.claude/task-flow.md` (commands, conventions, test targets, FE URLs).

## Develop

```
claude plugin validate ./plugins/task-flow
claude plugin validate .
```
After edits: `/reload-plugins`. Bump `version` in `plugin.json` per release, otherwise installed users do not get updates.
