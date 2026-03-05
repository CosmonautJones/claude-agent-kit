# AGENTS.md — Lean Agile Team

> **Universal agent instructions.** This file is the single source of truth for Claude Code, Codex, and Cursor. Cursor-specific rules live in `.cursor/rules/*.mdc`. See [Cross-Tool Compatibility](#cross-tool-compatibility) at the bottom.

This project uses a multi-agent development system. Seven specialized roles coordinate through shared task lists to ship features, fix bugs, refactor code, run tests, and maintain documentation.

## Agents

| Agent | Role | Capabilities | Key Traits |
|-------|------|-------------|------------|
| **full-stack-developer** | C#, VB.NET, COBOL (Fujitsu), .NET interop | Read, write, edit files; run commands; search code | Stack-adaptive, isolated workspace, project memory |
| **database-admin** | COBOL data layer, .NET interop, SQL Server, data integrity | Read, write, edit files; run commands; search code | Stack-adaptive, isolated workspace, project memory |
| **shipper** | Git, testing, building, deployment, PRs | Run commands; read and search files | Pipeline owner, full automation access |
| **reviewer** | Security, bugs, performance review | Read and search files only | Read-only, non-blocking except security |
| **documentor** | Documentation creation and maintenance | Read, write, edit files; run commands; search code | Runs after tests pass |
| **meta-agent** | Generate new custom agents | Write, read, search files; fetch web docs | On-demand agent creation |
| **meta-skills-agent** | Generate new workflow skills | Read, write files; fetch web docs; search code | On-demand skill creation |

### full-stack-developer

Feature implementation across the Global Shop Solutions stack. Detects project context by scanning `.sln`, `.csproj`, `.vbproj`, COBOL copybooks (`.cpy`), and `.cob`/`.cobol` source files.

**Primary languages:** C#, VB.NET, COBOL (Fujitsu NetCOBOL)

**Responsibilities:**
- C# and VB.NET application development (.NET Framework / .NET 6+)
- COBOL program maintenance and new development (Fujitsu NetCOBOL)
- Interop between .NET managed code and COBOL modules
- Windows Forms, WPF, or web UI following project conventions
- Minimal tests for critical paths only (data integrity, business logic, interop boundaries)

**Boundaries:** Does not modify the COBOL data layer or copybook definitions directly (delegates to database-admin). Does not push to main without review.

**Tech-stack skills available:** `csharp-dotnet`, `vbnet-patterns`, `cobol-fujitsu`, `dotnet-cobol-interop`

### database-admin

Data layer design, optimization, and implementation. Specializes in the hybrid COBOL + OOP + .NET data layer architecture used at Global Shop Solutions.

**Responsibilities:**
- COBOL data layer programs — file I/O, record layouts, copybook definitions (`.cpy`)
- OOP wrappers that bridge COBOL data access with .NET (C#/VB.NET interop)
- Database schema changes and stored procedures (SQL Server)
- Data integrity, indexing, and query optimization
- Minimal tests for data integrity and interop boundaries only

**Protection rules:**
- Never modify production copybooks or data files without explicit user approval
- Never deploy COBOL changes to production without explicit user approval
- Always validate record layouts against existing copybooks before changes
- Always work locally first

### shipper

Owns the entire release pipeline. No other agent should execute git commands.

**Responsibilities:**
- Branch creation (`feature/`, `hotfix/`, `refactor/`, `test/`)
- Atomic commits with conventional messages (`feat:`, `fix:`, `refactor:`, `test:`)
- Test suite execution and result reporting
- Building, deployment, and PR creation
- Merges and release tagging

**Conventions:** Never force-push to main. Feature branches for all development. One logical change per commit.

### reviewer

Pragmatic, high-impact code reviews. Focuses on what matters and skips style nitpicks.

**Review categories:**
- **CRITICAL (blocking):** Security vulnerabilities, data loss risks — must fix before deploy
- **WARNING (non-blocking):** Bugs and performance issues — should fix soon
- **NOTE (informational):** Improvement suggestions — fix when convenient

**Verdict options:** `APPROVE`, `APPROVE_WITH_WARNINGS`, `REQUEST_CHANGES`

Only blocks deployment for security issues. Operates in read-only plan mode.

### documentor

Creates, maintains, and organizes codebase documentation in `docs/`.

**Capabilities:**
- Full documentation generation (architecture, API, guides, reference)
- Incremental updates based on `git diff`
- Cross-linked documents with table of contents
- kebab-case file naming

### meta-agent

Generates new agent configuration files from a user description. Analyzes requirements, selects appropriate tools and settings, and writes a complete agent definition file to `agents/`.

### meta-skills-agent

Generates new workflow skill files from a user description. Reads existing skill patterns for reference, determines agent involvement and task dependencies, and writes a complete skill definition file.

---

## Workflow Skills

### `/team-ship` — Build and Deploy Features

Branch, implement, review, test, deploy, and PR. Balanced speed.

```text
Shipper ──► Full Stack Dev + DB Admin (parallel) ──► Shipper ──► Reviewer ──► Shipper ──► Documentor ──► Shipper
Branch       Implement feature                       Commit       Review       Test         Update Docs    Deploy+PR
```

| # | Task | Owner | Blocked By |
|---|------|-------|------------|
| 1 | Create feature branch `feature/<name>` from main | shipper | — |
| 2 | Implement the feature end-to-end | full-stack-developer | 1 |
| 3 | Implement database/data layer changes (if needed) | database-admin | 1 |
| 4 | Commit all changes with conventional message | shipper | 2, 3 |
| 5 | Review implementation for security, bugs, performance | reviewer | 4 |
| 6 | Run full test suite | shipper | 5 |
| 7 | Fix regressions (if tests fail — loop back to 6) | full-stack-developer | 6 |
| 8 | Update documentation for feature changes | documentor | 6 |
| 9 | Deploy and create PR to main | shipper | 6, 8 |

**Parallelism:** Tasks 2 and 3 run simultaneously. If task 6 finds failures, task 7 fixes them and loops back.

---

### `/team-fix` — Emergency Bug Fixes

Rapidly diagnose and fix production issues. No reviewer step — speed is the priority.

```text
Shipper ──► Full Stack Dev + DB Admin (parallel) ──► Shipper ──► Shipper ──► Documentor ──► Shipper
Hotfix       Diagnose & patch                        Commit      Test+Deploy  Update Docs    Merge
```

| # | Task | Owner | Blocked By |
|---|------|-------|------------|
| 1 | Create hotfix branch `hotfix/<issue-id>` from main | shipper | — |
| 2 | Diagnose root cause and implement minimal fix | full-stack-developer | 1 |
| 3 | Fix data layer issues (if applicable) | database-admin | 1 |
| 4 | Commit fix with message `fix: <description>` | shipper | 2, 3 |
| 5 | Run focused tests and deploy | shipper | 4 |
| 6 | Update documentation if fix affects docs | documentor | 5 |
| 7 | Merge hotfix to main, tag patch release | shipper | 5, 6 |

**Parallelism:** Tasks 2 and 3 run simultaneously if both needed.

---

### `/team-cleanup` — Technical Debt and Refactoring

Analyze first, then refactor. Reviewer-led: analyze before changing.

```text
Shipper ──► Reviewer ──► Full Stack Dev + DB Admin (parallel) ──► Shipper ──► Shipper ──► Documentor ──► Shipper
Branch       Analyze       Refactor                                Commit      Test         Update Docs     PR
```

| # | Task | Owner | Blocked By |
|---|------|-------|------------|
| 1 | Create branch `refactor/<area>` from main | shipper | — |
| 2 | Analyze code for smells, bottlenecks, and refactoring opportunities | reviewer | 1 |
| 3 | Refactor application code based on review findings | full-stack-developer | 2 |
| 4 | Refactor data layer based on review findings (if needed) | database-admin | 2 |
| 5 | Commit refactoring changes | shipper | 3, 4 |
| 6 | Run full test suite and validate | shipper | 5 |
| 7 | Update documentation for refactoring changes | documentor | 6 |
| 8 | Create PR to main with before/after summary | shipper | 6, 7 |

**Parallelism:** Tasks 3 and 4 run simultaneously after review completes.

---

### `/team-run-tests` — Batch Test and Fix

Run all tests, batch-fix failures, loop until green.

```text
Shipper ──► Shipper ──► Full Stack Dev + DB Admin ──► Shipper ──► Shipper ──► Reviewer ──► Documentor ──► Shipper
Branch       Run tests    Fix failures (parallel)      Commit      Re-test     Review       Update Docs     PR
                              ↑                                       │
                              └───────── loop if still failing ───────┘
```

| # | Task | Owner | Blocked By |
|---|------|-------|------------|
| 1 | Create branch `test/<timestamp>` from main | shipper | — |
| 2 | Run full test suite, compile failure report | shipper | 1 |
| 3 | Fix application test failures | full-stack-developer | 2 |
| 4 | Fix data layer test failures | database-admin | 2 |
| 5 | Commit all fixes | shipper | 3, 4 |
| 6 | Re-run previously failed tests | shipper | 5 |
| 7 | Run full validation suite | shipper | 6 |
| 8 | Review all fixes for quality | reviewer | 7 |
| 9 | Update documentation if fixes affect docs | documentor | 8 |
| 10 | Create PR to main with test report | shipper | 8, 9 |

**Parallelism:** Tasks 3 and 4 run simultaneously. Loop: if task 6 finds remaining failures, create new fix tasks and repeat.

---

### `/team-add-tests` — Critical Test Coverage

Test the 20% that prevents 80% of disasters. Time-boxed effort.

```text
Shipper ──► Reviewer ──► Full Stack Dev + DB Admin (parallel) ──► Shipper ──► Documentor ──► Shipper
Branch       Identify       Write Tests                           Run Tests    Update Docs    Commit+PR
             Critical Gaps
```

| # | Task | Owner | Blocked By |
|---|------|-------|------------|
| 1 | Create branch `test/critical-coverage-<area>` from main | shipper | — |
| 2 | Identify CRITICAL untested code paths | reviewer | 1 |
| 3 | Write minimal tests for critical application paths | full-stack-developer | 2 |
| 4 | Write minimal tests for critical data integrity paths (if needed) | database-admin | 2 |
| 5 | Run full test suite including new tests | shipper | 3, 4 |
| 6 | Update documentation for new test coverage | documentor | 5 |
| 7 | Commit and create PR to main | shipper | 5, 6 |

**Critical = code that could:** break auth, lose/corrupt data, break payments, cause outages.
**Skip:** UI formatting, nice-to-have features, edge cases that won't impact production.

---

## Utility Skills

| Skill | Purpose | Agents Used |
|-------|---------|-------------|
| `/team-init-docs [area]` | Generate comprehensive codebase documentation | documentor |
| `/team-update-docs [changes]` | Update docs to reflect recent code changes | documentor |
| `/team-create-agent <description>` | Create a new custom agent via meta-agent | meta-agent |
| `/team-create-skill <description>` | Create a new workflow skill via meta-skills-agent | meta-skills-agent |
| `/team-repo-status [focus]` | Repository health report (git, todos, activity) | — (direct execution) |
| `/team-audit [action]` | Audit log analysis (summary, report, metrics, timeline, verify, anomalies) | — (direct execution) |
| `/team-all-tools` | List all available tools with signatures | — (direct execution) |

---

## Tech-Stack Skills

Referenced automatically by full-stack-developer and database-admin when the matching stack is detected:

| Skill | Patterns Covered |
|-------|-----------------|
| `csharp-dotnet` | C# conventions, .NET Framework / .NET 6+, WinForms, WPF, ASP.NET, dependency injection |
| `vbnet-patterns` | VB.NET idioms, legacy modernization, .NET interop, Option Strict/Explicit conventions |
| `cobol-fujitsu` | Fujitsu NetCOBOL syntax, copybooks, file I/O, PERFORM/EVALUATE patterns, paragraph structure |
| `dotnet-cobol-interop` | Managed/unmanaged interop, P/Invoke, COM wrappers, data marshalling between .NET and COBOL |

---

## Quality Gates (Hooks)

Three hooks enforce quality and provide observability:

### TaskCompleted — Completion Verification

**Type:** Prompt | **Event:** Any task completes

Before marking a task as completed, the hook verifies:
1. All acceptance criteria from the task description are met
2. No partial implementations or TODO placeholders remain
3. If code was changed, it compiles/runs without errors

Rejects completion and explains what remains if criteria are not met.

### TeammateIdle — Activity Logging

**Type:** Command | **Event:** Any teammate becomes idle

Logs idle events to `<tmpdir>/lean-agile-team.log` with timestamp and agent name. Used for observability and debugging team coordination.

### SubagentStop (shipper) — Pipeline Integrity

**Type:** Prompt | **Event:** Shipper agent finishes | **Scope:** `agent_name: "shipper"` only

Validates git state when the shipper finishes:
1. No uncommitted changes (`git status` is clean)
2. No failed pushes or unresolved merge conflicts
3. Current branch is in a consistent state

---

## Coordination Model

### Parallel Execution (recommended)

When the AI tool supports multi-agent or parallel execution:

- Agents work in parallel where workflows allow (e.g., full-stack-developer + database-admin)
- Shared task lists with dependency tracking
- Tasks specify owners and blocked-by dependencies

### Sequential Fallback

When parallel execution is not available:

- Execute agents sequentially in the order specified by each workflow
- Same workflow steps and quality gates
- Slightly slower due to serial execution

### Conventions

- **Branching:** `feature/<name>`, `hotfix/<id>`, `refactor/<area>`, `test/<scope>`
- **Commits:** Conventional format — `feat:`, `fix:`, `refactor:`, `test:`, `docs:`
- **Reviews:** Non-blocking except for CRITICAL security issues
- **Testing:** Minimal — test only what could break production
- **Shipping:** Working software over perfect software

---

## Cross-Tool Compatibility

This `AGENTS.md` is the single source of truth for all three supported tools:

| File | Tool | How It Works |
|------|------|-------------|
| `AGENTS.md` | Claude Code, Codex | Both auto-discover `AGENTS.md` at the repo root |
| `.cursor/rules/*.mdc` | Cursor | Modular rules with glob-based activation |

- **Claude Code** reads `AGENTS.md` as project-level agent instructions automatically.
- **Codex** (OpenAI) reads `AGENTS.md` from the repo root as its instruction file.
- **Cursor** uses `.cursor/rules/*.mdc` files which contain the same workflows and conventions in Cursor's native format.

When updating agent behavior, edit this `AGENTS.md` first, then sync the `.cursor/rules/` files.
