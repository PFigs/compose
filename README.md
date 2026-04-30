# compose

A Jazz-themed planning and execution skill for Claude Code. Compose runs a small ensemble of named agents that plan, test, implement, and review work end-to-end, with TDD (red-green-refactor) and SOLID baked in as non-negotiable disciplines.

## The ensemble

| Name | Role |
|------|------|
| Ella | Orchestrator |
| Bird | Codebase explorer |
| Duke | Architect / planner |
| Monk | Test writer (writes failing gates first) |
| Hawk | Implementer (turns gates green) |
| Django | Spec-compliance reviewer |
| Billie | Code-quality reviewer |
| Count | Dispatcher for parallel work |

## Install

Clone into your Claude Code skills directory:

```bash
git clone git@github.com:PFigs/compose.git ~/.claude/skills/compose
```

Install the review tool used during plan/spec review:

```bash
uv tool install redliner
```

## Use

Trigger via `/compose` or phrases like "let's plan this" / "design this". At session start, compose inventories the other skills available in your environment and asks once which to wire into each phase (planning, tests, implementation, review, wrap-up). It uses what you pick and falls back to its built-in flow for any phase you leave unassigned.

## Layout

```
SKILL.md         orchestration logic for Ella
prompts/         agent prompts (hawk, billie, monk, django)
```
