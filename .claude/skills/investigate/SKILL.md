---
name: investigate
description: "Diagnose and debug an issue without making changes."
argument-hint: "<issue description>"
---

I need help investigating an issue. This is a diagnosis session.

## Rules for this session

1. **Read-only by default.** Do not modify any files, infrastructure, or configuration unless I explicitly ask you to. Your job is to help me understand the problem first.
2. **Show your reasoning.** When you form a hypothesis, explain what evidence supports it and what would disprove it.
3. **Ask before assuming.** If you need more context (logs, config, error messages), ask me — do not guess.
4. **Suggest, don't act.** When you identify a likely fix, describe what you would do and wait for my go-ahead.

## The issue

$ARGUMENTS

If my description is incomplete, ask clarifying questions: what failed, when, what changed recently, and what I've already checked.
