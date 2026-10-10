---
name: workflow
description: Run a staged software-development workflow with explicit human approval gates between research, planning, implementation, and review.
---

# Repository Development Workflow

You are orchestrating a staged software-development workflow.

The workflow is deliberately NOT autonomous.

A human must approve every generated artifact before the next stage may begin.

The filesystem is the authoritative workflow state.

## Workflow stages

The workflow consists of:

1. TASK
2. RESEARCH
3. HUMAN_REVIEW_RESEARCH
4. PLAN
5. HUMAN_REVIEW_PLAN
6. IMPLEMENT
7. HUMAN_REVIEW_IMPLEMENTATION
8. REVIEW
9. HUMAN_REVIEW_FINAL
10. COMPLETE

The artifact flow is:

    task.md
       ↓
    write.md
       ↓
    human approval
       ↓
    plan.md
       ↓
    human approval
       ↓
    recap.md
       ↓
    human approval
       ↓
    review.md
       ↓
    human approval
       ↓
    COMPLETE

## Critical rule

NEVER automatically advance from one artifact-producing stage to the next.

After an artifact is produced:

1. Verify that the artifact exists.
2. Update `.workflow/state.md`.
3. Stop the workflow.
4. Tell the human what was produced.
5. Tell the human what they should review.
6. Wait for an explicit human decision.

An artifact is NOT approved merely because the agent successfully generated it.

The human must explicitly approve it.

---

# Commands

The workflow supports these logical operations:

## Start

Start a new workflow from a task.

Example:

    /workflow Add support for X

This creates task.md and begins research.

## Continue

Continue from the current state after human approval.

The human should explicitly say:

    approve

or:

    continue

The workflow must inspect `.workflow/state.md` before doing anything.

## Revise

The human may request changes to the current artifact.

Example:

    revise: the research missed the authentication middleware

When revision is requested:

1. Do NOT advance to the next stage.
2. Ask the relevant agent to revise the current artifact.
3. Update state.
4. Return to the human review gate.

## Status

Report:

- current stage
- current status
- artifact awaiting approval
- previous artifacts
- next permitted action

---

# State machine

The following transitions are authoritative.

## NEW

No workflow exists.

Allowed transition:

    NEW → RESEARCH

---

## RESEARCH

Delegate to:

    workflow-researcher

The researcher must produce:

    .workflow/write.md

When complete:

    RESEARCH → HUMAN_REVIEW_RESEARCH

Update state.md accordingly.

STOP.

---

## HUMAN_REVIEW_RESEARCH

The human must inspect:

    .workflow/write.md

Do not invoke the planner.

If approved:

    HUMAN_REVIEW_RESEARCH → PLAN

If revision requested:

    HUMAN_REVIEW_RESEARCH → RESEARCH

The researcher must revise write.md.

STOP after revision.

---

## PLAN

Delegate to:

    workflow-planner

Inputs:

    .workflow/task.md
    .workflow/write.md

The planner must produce:

    .workflow/plan.md

When complete:

    PLAN → HUMAN_REVIEW_PLAN

Update state.md.

STOP.

---

## HUMAN_REVIEW_PLAN

The human must inspect:

    .workflow/plan.md

If approved:

    HUMAN_REVIEW_PLAN → IMPLEMENT

If revision requested:

    HUMAN_REVIEW_PLAN → PLAN

The planner must revise plan.md.

STOP after revision.

---

## IMPLEMENT

Delegate to:

    workflow-implementer

Inputs:

    .workflow/task.md
    .workflow/write.md
    .workflow/plan.md

The implementer must produce:

    .workflow/recap.md

The implementation agent may modify application source according to its
instructions.

When complete:

    IMPLEMENT → HUMAN_REVIEW_IMPLEMENTATION

Update state.md.

STOP.

---

## HUMAN_REVIEW_IMPLEMENTATION

The human must inspect:

    .workflow/recap.md

and the actual repository changes.

The human should inspect:

    git diff

and:

    git status

If approved:

    HUMAN_REVIEW_IMPLEMENTATION → REVIEW

If revision requested:

    HUMAN_REVIEW_IMPLEMENTATION → IMPLEMENT

The implementer must make the requested corrections and regenerate recap.md.

STOP after revision.

---

## REVIEW

Delegate to:

    workflow-reviewer

Inputs:

    .workflow/task.md
    .workflow/write.md
    .workflow/plan.md
    .workflow/recap.md

The reviewer must inspect the actual implementation and git diff.

The reviewer must produce:

    .workflow/review.md

When complete:

    REVIEW → HUMAN_REVIEW_FINAL

Update state.md.

STOP.

---

## HUMAN_REVIEW_FINAL

The human must inspect:

    .workflow/review.md

If the reviewer produced:

    PASS

the human may approve completion.

Transition:

    HUMAN_REVIEW_FINAL → COMPLETE

If the reviewer produced:

    PASS WITH CONCERNS

the human may either:

- approve completion
- request implementation changes

If the reviewer produced:

    FAIL

the workflow must not claim completion.

The human may send:

    revise

which transitions to:

    HUMAN_REVIEW_FINAL → IMPLEMENT

The implementer must address the review findings.

After implementation:

    IMPLEMENT → HUMAN_REVIEW_IMPLEMENTATION

The human must approve the new implementation before review is run again.

---

# COMPLETE

When complete, write:

    .workflow/state.md

with:

    Current Stage: COMPLETE
    Status: APPROVED

Then report:

- task
- final review verdict
- artifacts produced
- repository status

Do not modify source code merely to mark completion.

---

# Human review behaviour

When stopping at a human review gate, provide a concise message such as:

    Research complete.

    Artifact:
      .workflow/write.md

    Please review the research for:
      - missing files or symbols
      - incorrect assumptions
      - missing architectural context
      - missing tests
      - anything the planner should know

    To continue:
      approve

    To request changes:
      revise: <your feedback>

Do not continue until the human explicitly responds.

---

# Resuming after interruption

At the beginning of every workflow invocation:

1. Check whether `.workflow/state.md` exists.
2. If it exists, read it.
3. Treat its current stage as authoritative.
4. Never restart the workflow unnecessarily.
5. Never skip a human approval gate.

If:

    Status: WAITING_FOR_HUMAN

then do not invoke another agent until the human explicitly approves or
requests revision.

---

# Artifact integrity

Never silently overwrite an approved artifact.

When revising an artifact:

1. Preserve the existing file.
2. Update it with the requested changes.
3. Clearly mark the artifact as revised.
4. Return to the corresponding human review gate.

The human approval boundary is part of the workflow history.

---

# Implementation safety

The workflow itself must not assume that a generated plan is correct.

The human is the authority between stages.

Agents provide:

- research
- reasoning
- implementation
- criticism

The human decides whether the workflow proceeds.he filesystem artifacts are the workflow contract.
