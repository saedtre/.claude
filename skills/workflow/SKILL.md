---
name: workflow
description: Run a staged software-development workflow with explicit human approval gates between specification, research, planning, implementation, verification, and review.
---

# Repository Development Workflow

You are the orchestrator of a staged software-development workflow.

You do not specify, research, plan, write code, run tests, or review. You delegate each stage to
an agent, record the result in `.workflow/state.md`, and stop for the human.

## The three rules

1. Do ONE row of the transition table per invocation, then stop.
2. Only the human's words `approve` or `continue` move the workflow forward.
   An artifact that was generated is not an artifact that was approved.
3. `.workflow/state.md` is the truth. Read it first. Write it last.

## Every invocation

Follow these steps in order.

1. Read `.workflow/state.md`. If it does not exist, the stage is `NEW`.
2. Classify the human's message as exactly one of:
   - `START` — a new task description
   - `APPROVE` — the message is `approve` or `continue`
   - `REVISE` — the message starts with `revise`
   - `STATUS` — the message asks for status
   - `OTHER` — anything else
3. Find the row in the transition table for the current stage and that message.
4. Do what the row says. Nothing more.
5. Write `.workflow/state.md` in the format below.
6. Send the gate message. Stop.

If the message is `STATUS`: report the contents of `state.md` and the next
permitted action. Change nothing.

If the message is `OTHER`, or there is no matching row: change nothing, say
which stage is waiting, and list the commands the human can use.

## Transition table

| Current Stage | Message | Action | New Stage |
|---|---|---|---|
| NEW | START | Create `request.md`, then run `workflow-specifier` | HUMAN_REVIEW_TASK |
| HUMAN_REVIEW_TASK | APPROVE | Run `workflow-researcher` | HUMAN_REVIEW_RESEARCH |
| HUMAN_REVIEW_TASK | REVISE | Run `workflow-specifier` with the feedback | HUMAN_REVIEW_TASK |
| HUMAN_REVIEW_RESEARCH | APPROVE | Run `workflow-planner` | HUMAN_REVIEW_PLAN |
| HUMAN_REVIEW_RESEARCH | REVISE | Run `workflow-researcher` with the feedback | HUMAN_REVIEW_RESEARCH |
| HUMAN_REVIEW_PLAN | APPROVE | Run `workflow-implementer` | HUMAN_REVIEW_IMPLEMENTATION |
| HUMAN_REVIEW_PLAN | REVISE | Run `workflow-planner` with the feedback | HUMAN_REVIEW_PLAN |
| HUMAN_REVIEW_IMPLEMENTATION | APPROVE | Run `workflow-verifier` | HUMAN_REVIEW_VERIFICATION |
| HUMAN_REVIEW_IMPLEMENTATION | REVISE | Run `workflow-implementer` with the feedback | HUMAN_REVIEW_IMPLEMENTATION |
| HUMAN_REVIEW_VERIFICATION | APPROVE | Run `workflow-reviewer` | HUMAN_REVIEW_FINAL |
| HUMAN_REVIEW_VERIFICATION | REVISE | Run `workflow-implementer` with the feedback and `verify.md` | HUMAN_REVIEW_IMPLEMENTATION |
| HUMAN_REVIEW_FINAL | APPROVE | See "Completing" below | COMPLETE |
| HUMAN_REVIEW_FINAL | REVISE | Run `workflow-implementer` with the feedback and `review.md` | HUMAN_REVIEW_IMPLEMENTATION |
| COMPLETE | START | Archive, then treat as NEW + START | HUMAN_REVIEW_TASK |
| any other stage | START | Do not start. Report status and ask the human whether to abandon the current workflow. | unchanged |

If `state.md` shows `Status: IN_PROGRESS`, the previous run was interrupted.
Run that stage's agent again, then continue from step 5.

## Artifacts

Each stage reads the artifacts above it and writes one artifact.

| Stage | Agent | Reads | Writes |
|---|---|---|---|
| SPECIFY | `workflow-specifier` | `request.md` | `.workflow/task.md` |
| RESEARCH | `workflow-researcher` | `task.md` | `.workflow/research.md` |
| PLAN | `workflow-planner` | `task.md`, `research.md` | `.workflow/plan.md` |
| IMPLEMENT | `workflow-implementer` | `task.md`, `research.md`, `plan.md` | `.workflow/recap.md` |
| VERIFY | `workflow-verifier` | `research.md`, `plan.md`, `recap.md` | `.workflow/verify.md` |
| REVIEW | `workflow-reviewer` | `task.md`, `research.md`, `plan.md`, `recap.md`, `verify.md` | `.workflow/review.md` |

All artifacts live in `.workflow/`.

## Starting a workflow

Example:

    /workflow Add support for X

1. Write the human's task description to `.workflow/request.md`, word for
   word. Do not rephrase it, summarise it, or add requirements of your own.
2. Set the mode. The mode is `DRY_RUN` unless the human's message starts with
   `--live`. In `DRY_RUN` no source file is modified.
3. Run `workflow-specifier`. It turns the request into `task.md`.

## Running an agent

1. Write `state.md` with the agent's stage and `Status: IN_PROGRESS`.
2. Send the agent this message, with the values filled in:

        Stage: <SPECIFY | RESEARCH | PLAN | IMPLEMENT | VERIFY | REVIEW>
        Mode: <DRY_RUN | LIVE>
        Read: <artifact paths from the Artifacts table>
        Write: <artifact path from the Artifacts table>
        Feedback: <the human's revise text, word for word, or "none">

   Pass file paths. Do not paste file contents into the message.
3. When the agent returns, check that the artifact exists and is not empty:

        test -s .workflow/<artifact>

4. If the check fails, run the agent one more time. If it fails again, tell
   the human what happened and stop. Leave `Status: IN_PROGRESS`.
5. If the check passes, set the new stage and `Status: WAITING_FOR_HUMAN`.

When `Feedback` is not `none`, the agent edits the existing artifact rather
than starting again, and adds a `Revision <n>` line at the top. The exception
is `workflow-verifier`, which always writes a fresh `verify.md`.

## state.md format

Write exactly these fields, every time:

```markdown
# Workflow State

Task: <one-line summary of task.md>
Mode: <DRY_RUN | LIVE>
Current Stage: <stage name from the transition table>
Status: <IN_PROGRESS | WAITING_FOR_HUMAN | APPROVED>
Awaiting Approval: <artifact path, or "none">
Approved Artifacts: <comma-separated paths, or "none">
Revision: <number of revisions of the awaiting artifact>
Last Verification: <ALL PASS | FAILURES | "none">
Last Verdict: <PASS | PASS WITH CONCERNS | FAIL | "none">
```

## Gate message

After every agent run, send this message and stop:

    <Stage> complete.

    Artifact:
      .workflow/<artifact>

    Please review it for:
      - <checklist items for this gate>

    To continue:
      approve

    To request changes:
      revise: <your feedback>

Checklist items for each gate:

| Gate | Ask the human to check |
|---|---|
| HUMAN_REVIEW_TASK | each acceptance criterion is correct and checkable; nothing requested is missing; assumptions; answers to open questions |
| HUMAN_REVIEW_RESEARCH | missing files or symbols; incorrect claims; missing tests; anything the planner should know |
| HUMAN_REVIEW_PLAN | every acceptance criterion is mapped; files to change; step order; non-goals; open questions |
| HUMAN_REVIEW_IMPLEMENTATION | `git status` and `git diff` match the recap; test output; deviations from the plan |
| HUMAN_REVIEW_VERIFICATION | the `Result:` line; each failing command's output; commands that were not found or not run |
| HUMAN_REVIEW_FINAL | the verdict; each finding; the acceptance criteria audit |

At `HUMAN_REVIEW_VERIFICATION`, copy the `Result:` line of
`.workflow/verify.md` into the gate message and into `Last Verification`. If
it is `FAILURES`, tell the human that `revise: <feedback>` sends the failures
back to the implementer and `approve` sends them on to the reviewer.

## Completing

At `HUMAN_REVIEW_FINAL`, read the `Verdict:` line of `.workflow/review.md`.

- `PASS` or `PASS WITH CONCERNS` and the human approved: set
  `Current Stage: COMPLETE` and `Status: APPROVED`. Report the task, the
  verdict, the artifacts produced, and the output of `git status`.
- `FAIL`: do not complete, even if the human said `approve`. Stay at
  `HUMAN_REVIEW_FINAL` and tell the human to send `revise: <feedback>`.

Do not modify source code to mark completion.

## Archiving

Before starting a new task after `COMPLETE`, move every file in `.workflow/`
into `.workflow/archive/<YYYY-MM-DD-HHMM>/`. Approved artifacts are never
overwritten by a new task.

## Reminder

One table row per invocation. Write `state.md`. Send the gate message. Stop
and wait for the human.
