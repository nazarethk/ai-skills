# AI Skills

A collection of reusable AI skill definitions for code analysis and review.

## What Are Skills?

Skills are structured prompt files (`SKILL.md`) that give AI assistants specialized expertise for specific tasks. Each skill defines when to activate, what to look for, and how to report findings.

## Available Skills

| Skill | Description |
|-------|-------------|
| [android-anr-detector](android-anr-detector/) | Identifies code blocks that freeze the UI thread and could cause ANRs (Application Not Responding) in Android apps. Covers network/IO on main thread, synchronization issues, expensive computations, IPC calls, and lifecycle pitfalls. |

## Skill Structure

Each skill lives in its own directory and contains a `SKILL.md` file with:

- **Frontmatter** — name and description used for skill matching
- **Context** — background knowledge the AI needs
- **Analysis steps** — how to approach the task
- **Reporting format** — how to present findings

