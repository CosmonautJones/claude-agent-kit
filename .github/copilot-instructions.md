# Copilot Custom Instructions

> This file adapts the project's universal `AGENTS.md` for GitHub Copilot. See `AGENTS.md` at the repository root for the full specification.

## Project Overview

This project uses a multi-agent development system with seven specialized roles: full-stack-developer, database-admin, shipper, reviewer, documentor, meta-agent, and meta-skills-agent.

## Development Principles

- Ship fast, learn faster — working software over perfect software
- Make it work first, optimize later
- Minimal testing — test only the 20% that prevents 80% of disasters (auth, payments, data loss, outages)
- Non-blocking reviews — only block deployment for CRITICAL security issues
- One logical change per commit using conventional commit format

## Conventions

- **Branching:** `feature/<name>`, `hotfix/<id>`, `refactor/<area>`, `test/<scope>`
- **Commits:** `feat:`, `fix:`, `refactor:`, `test:`, `docs:`, `chore:`
- **File naming:** kebab-case for documentation files
- Never force-push to main. Feature branches for all development.

## Stack Detection

On first action, detect the tech stack by scanning config files (`package.json`, `requirements.txt`, `go.mod`, `Gemfile`, `Cargo.toml`, `composer.json`) and tailor implementation to the detected stack.

## Code Review Standards

When reviewing code, focus on high-impact issues only:
- **CRITICAL (blocking):** Security vulnerabilities (OWASP top 10), data loss risks
- **WARNING (non-blocking):** Bugs, performance issues (N+1 queries, missing indexes, memory leaks)
- **NOTE (informational):** Improvement suggestions
- Skip style nitpicks and formatting

## Database Safety

- Never run destructive database commands (`DROP`, `TRUNCATE`, `DELETE` without `WHERE`, `reset`, `flush`) without explicit approval
- Never deploy to production databases without explicit approval
- Always work with local databases first

## Quality Gates

Before completing any task:
1. All acceptance criteria are met
2. No partial implementations or TODO placeholders remain
3. Code compiles/runs without errors
4. Git state is clean after pipeline operations
