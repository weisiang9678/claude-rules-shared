---
description: "Start a structured planning session — explores requirements and design before implementation."
argument-hint: "<objective or topic to plan>"
---

# Planning Session

Structured planning mode for designing data platform changes before implementation.

<HARD-GATE>
Do NOT invoke any implementation skill, execute any commands, generate application code, or create infrastructure files until you have presented a plan and the user has explicitly approved it. This applies to EVERY task regardless of perceived simplicity. When the plan is approved, your ONLY next action is to save the plan and tell the user to start a new session with `/implement`. You do NOT start implementing.
</HARD-GATE>

### Anti-Pattern: "This Is Too Simple To Need A Plan"

Every task goes through this process. A config change, a single table addition, a one-line pipeline fix. The plan can be short (a few sentences for truly simple tasks), but it must exist and be approved. "Simple" tasks are where unexamined assumptions cause the most wasted work.

## Behavior Rules

This skill changes how Claude operates for the duration of the session.

### Do

- Read and use the plan template from [plan-template.md](references/plan-template.md) as the output structure
- Fill in sections **only** as the user provides information — ask for what is missing
- Propose options and trade-offs when there are meaningful alternatives
- Update the plan iteratively as the discussion progresses
- Produce a single, clean markdown plan as the final deliverable

### Do Not

- Execute commands, generate application code, or create infrastructure files
- Fill in sections with assumptions or placeholders the user has not confirmed
- Proceed to implementation — that happens in a separate session with the approved plan
- Skip sections silently — flag them as open questions instead

## Procedure

If invoked with arguments, use them as the initial objective. If invoked without arguments, ask the user to describe what they want to achieve.

### Step 1: Load Template

Read `references/plan-template.md` and use it as the skeleton for the plan.

### Step 2: Clarify Objective

Ask the user to describe what they want to achieve. Fill in **§1 Objective** and **§2 Context & Constraints** based on their input. Record the selected approach in **§3 Alternatives Considered**. Ask about:
- Target GCP project(s) and environment(s)
- Key services involved
- Upstream/downstream dependencies
- Known constraints or risks

### Step 3: Propose Approaches

Propose 2-3 different approaches with trade-offs. Lead with your recommendation and explain why. Present options conversationally. The user picks one before you fill in the rest of the plan.

If only one viable approach exists, state why and confirm the user agrees — don't fabricate alternatives.

### Step 4: Establish Current and Target State

Fill in **§4 Current State** and **§5 Target State**. If the user is unclear on either, help them articulate it before moving on.

### Step 5: Break Down into Phases

Decompose the work into phases in **§6 Phases & Tasks**. Each phase must have:
- A clear goal
- Concrete tasks with enough detail for implementation
- A **stop-gate** — during implementation, Claude pauses after each phase for the user's go/no-go

### Step 6: Define Validation and Rollback

Fill in **§7 Validation & Testing** (include the Phase column so checks can be scoped to phases during execution) and **§8 Rollback Plan**. These are non-optional — every plan must address how to verify success and how to undo changes.

### Step 7: Capture Open Items

Record unresolved decisions in **§9 Open Questions**. The plan is not ready for implementation until all open questions are resolved or explicitly deferred.

### Step 8: Deliver the Plan

Present the completed plan as a markdown file for the user to review.

## After Plan Approval

1. Save the plan to `plans/YYYY-MM-DD-<topic>.md`.
2. State: "Plan saved to `<path>`. To execute, run `/implement <path>` in a new session or this session."
3. **STOP.** Do not begin implementation. Do not offer to "get started." Do not generate code. The planning session is complete.

## Integration

- **Produces:** Plan document consumed by `implement` skill.
- **Next step:** `/implement <plan-path>` (invokes `implement`).
- **Related:** `verification-before-completion` is used during execution, not during planning.

## Reference Files

- [plan-template.md](references/plan-template.md) — Plan structure template
