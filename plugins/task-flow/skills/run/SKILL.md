---
name: run
description: Run a task through a strict sequential pipeline built on superpowers (plan, impact review, subagent implementation with review, verify, user-style test). Use when the user types /task-flow:run <task> or asks to run a task through the workflow agent.
disable-model-invocation: true
---

# task-flow

Orchestrator. YOU (main thread) run the stages one at a time. Subagents cannot spawn subagents, so never delegate orchestration. Never run two stages in parallel. Never skip a stage.

Planning, implementation and code review come from the `superpowers` plugin (required dependency). This plugin adds the three things it lacks: impact review, verify with project commands, and user-style testing with a BE approval gate.

Own agents: `task-flow:impact-reviewer`, `task-flow:verifier`, `task-flow:user-tester`.
Superpowers skills used: `superpowers:brainstorming`, `superpowers:writing-plans`, `superpowers:subagent-driven-development`, `superpowers:verification-before-completion`.

Task = the skill args. If empty, ask the user for the task.

## Setup
1. Slug: kebab-case, <=40 chars. Run dir: `.claude/runs/<slug>/` (create). Write `00-task.md` with the verbatim task and today's date. If `.gitignore` lacks `.claude/runs/`, tell the user (do not edit silently).
2. `git status`: if the tree is dirty with unrelated changes, tell the user and ask before continuing.
3. One todo item per stage.
4. If `.claude/task-flow.md` exists (commands, conventions, safe test targets), mention it in every agent prompt.
5. Check superpowers skills are available. If not, stop and tell the user to install: `/plugin install superpowers@claude-plugins-official`.

## Pipeline

| # | Stage | How | Gate to continue |
|---|-------|-----|------------------|
| 1 | Plan | Invoke `superpowers:brainstorming` only if the task is vague, then `superpowers:writing-plans`. | Plan file exists. Record its path in `00-task.md`. |
| 2 | Impact review | Spawn `task-flow:impact-reviewer` with run dir + plan path. | `02-impact.md` first line `APPROVE`. `REJECT` -> back to stage 1 (revise the plan with the notes), then stage 2 again. |
| 3 | Implement + review | Invoke `superpowers:subagent-driven-development` on the plan. It does per-task review (spec + quality) and a whole-branch review. | Its final review has no open Critical/Important issues. |
| 4 | Verify | Spawn `task-flow:verifier`, follow `superpowers:verification-before-completion` (evidence before claims). | `05-verify.md` `Verdict: PASS`. `FAIL` -> fix via stage 3 (a fix task), then stage 4 again. |
| 5 | User-style test | Pre-flight below, then spawn `task-flow:user-tester`. | `06-usertest.md` `Verdict: PASS`. `FAIL` -> fix via stage 3, then stages 4 and 5 again. |
| 6 | Summary | Write `07-summary.md`. | - |

Plan requirements (say this when invoking writing-plans): the plan must contain a **Scope tag** (`BE`, `FE`, or `BE+FE`), numbered **Acceptance criteria**, and a **Test scenarios** section (happy path, edge, permission/role per actor, validation errors, concurrency/idempotency, regression of one neighbouring feature; each with actor, input, expected result).

Prompt for each own agent: slug, run dir path, plan path, stage, which files to read, and (on loops) the exact failing items. Keep it short.

## Stage 5 pre-flight
Read the **Scope tag** in the plan. Run each part separately.
- **FE part**: no approval needed. Tell the agent `SCOPE=FE`, plus the FE URL if the plan/user/`.claude/task-flow.md` gave one (else ask once).
- **BE part**: ping the user first with AskUserQuestion: which target (local / test / uat, per `.claude/task-flow.md` if it lists them) and approval to run + write `tf_test_` data there. Only after approval spawn the agent with `SCOPE=BE` and `APPROVED TARGET=<answer>`. No answer or "skip" -> do not run; record `SKIPPED by user` in the summary. Prod is never an option.
- Ask once per run; reuse on later loops.

## Loop rules
- Any fix goes through stage 3, and every later stage reruns in order.
- Max 3 loops per gate. Still failing -> STOP, report latest findings. Do not weaken a gate, edit verdict files, or tell an agent to ignore findings.
- Read each output file yourself and check the verdict line. Do not trust the chat summary alone.
- A stage that reports BLOCKED: stop, show the user, ask how to proceed.

## Hard rules
- No commit, push, PR, merge or deploy unless the user asks afterwards. Do not run `superpowers:finishing-a-development-branch` unless asked. Follow the user's own commit rules when asked to commit.
- Never run against prod. Test/uat only with explicit approval at pre-flight.
- Reply in the user's language.

## Finish
`07-summary.md`: task, `git diff --stat`, verdict per stage, loops used, deviations, residual risks, manual follow-ups (migrations, env vars, deploy order).
Final message to user: 5-10 lines max, plus path to `07-summary.md`.
