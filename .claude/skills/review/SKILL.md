---
name: review
description: "Review code or architecture against project conventions and best practices."
argument-hint: "<files, PR, or area to review>"
---

I need a review. Check the following against our project rules and skills.

## Rules for this session

1. **Load relevant skills.** Before reviewing, read the skills that apply to what you're reviewing (e.g. `python-code-standards` for code, `deploy-cloudrun` for Cloud Run manifests, `pulumi-infrastructure` for IaC).
2. **Be specific.** Point to exact files and lines. Do not give generic advice.
3. **Categorize findings.** Use: 🔴 Must fix (bugs, convention violations, security) → 🟡 Should fix (maintainability, clarity) → 🟢 Nitpick (style preferences).
4. **No changes.** Do not modify any files. Provide findings as a summary I can act on.
5. **Acknowledge what's good.** If something is done well, say so briefly.

## What to review

$ARGUMENTS

If I haven't specified what to review, ask me: which files, which PR, or which area of the codebase.
