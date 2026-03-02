---
description: "Execute an approved plan phase by phase with verification gates."
argument-hint: "<path to approved plan file>"
---

# Executing Plans

Phase execution with verification gates. Follow the plan exactly — flag deviations, don't silently adjust.

## Procedure

### Step 1: Receive the Plan

Two paths:

- **File path provided** (via `/implement <path>`): Read the file.
- **No path provided** (bare `/implement`): Ask: "Which plan should I execute? Provide a file path, or paste the plan content." Wait for the user's response before proceeding.

### Step 2: Review Plan Critically

Read the plan end-to-end. Check:

- Are there ambiguities or gaps?
- Are phase-scoped verification checks defined in §7 (with Phase column)?
- Is the target GCP project and environment specified in §2?

If concerns → raise with user before starting.
If no concerns → confirm environment and proceed.

### Step 3: Execute Phase

For each task in the current phase: follow the plan exactly.

- Do not silently deviate.
- If you see a plan issue (missing step, incorrect command, outdated reference), flag it and wait for the user's decision.

### Step 4: Verify Phase

**REQUIRED: Use the `verification-before-completion` skill.**

Run ONLY the verification checks from §7 where the Phase column matches the current phase. Report actual command output.

If any check fails → report with evidence, do not proceed.

### Step 5: Report and Wait

Present:
- Tasks completed
- Verification output (actual results)
- Issues encountered

Say: "Ready for your go/no-go on Phase N+1."

Do not proceed without explicit go.

### Step 6: Continue or Complete

- **Go** → next phase (return to Step 3).
- **No-go** → wait for instructions.
- **All phases complete** → "All phases complete. Consider running `/review` against the plan to check for gaps."

## Blocker Handling

STOP immediately. Show the exact error (full output). Do not attempt a fix unless the user asks.

State: "I'm blocked on task X.Y. Here's the error: [error]. How would you like me to proceed?"

## Red Flags

- "I'll just adjust the plan slightly" → flag, don't adjust.
- "This step seems wrong but I'll work around it" → flag, don't work around.
- "I'll skip this verification since the previous one passed" → never skip.

## Integration

- **REQUIRED:** `verification-before-completion` at every phase boundary.
- **Consumes:** Plan document produced by `design` skill.
- **Related:** `review` skill (suggested after all phases complete).
