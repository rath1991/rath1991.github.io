---
layout: workshop
title: Reference — Agentic Coding Best Practices
permalink: /workshop/reference/
description: Canonical companion to the workshop. Full mental model, cross-platform framework, anti-patterns, model selection, and quick-reference cheatsheet for Claude Code and VS Code Copilot.
---

# Agentic Coding Best Practices
## Claude Code + VS Code Copilot Agents — From Project Planning to Shipping

> A comprehensive framework for solo developers building products with AI agents.
> Synthesized from Anthropic docs, Boris Cherny's workflow, and community research (April 2026).

Download: [agentic-coding-best-practices.md](https://raw.githubusercontent.com/rath1991/rath1991.github.io/main/workshop-reference.md) (raw markdown)

---

## The Mental Model: A Reliability Stack {#reliability-stack}

Think of your agentic setup as a **layered operating system**, not just a chatbot with tools:

<pre class="ascii-diagram">
┌─────────────────────────────────────────────────────────────┐
│  LAYER 5 — EVALUATOR / VERIFICATION                         │
│  verify-app subagent, tests, hooks, PostToolUse             │
├─────────────────────────────────────────────────────────────┤
│  LAYER 4 — SUBAGENTS / AGENT TEAMS                          │
│  Isolated workers for context-safe parallel execution       │
├─────────────────────────────────────────────────────────────┤
│  LAYER 3 — SKILLS / SLASH COMMANDS                          │
│  Repeatable workflows, domain knowledge, inner-loop tasks   │
├─────────────────────────────────────────────────────────────┤
│  LAYER 2 — PLANNING (Plan Mode / Plan Agent)                │
│  Spec → PRD → Architecture → Task List before any code      │
├─────────────────────────────────────────────────────────────┤
│  LAYER 1 — MEMORY / CONTEXT (CLAUDE.md / copilot-instructions)│
│  Project rules, conventions, anti-patterns, always-on context│
└─────────────────────────────────────────────────────────────┘
</pre>

**Key Insight (Boris Cherny):** "Coding becomes a pipeline of phases: spec, draft, simplify, verify. Each phase benefits from a different mind."

---

## Part 1: CLAUDE CODE {#part-1-claude-code}

### 1.1 — Memory Layer: CLAUDE.md {#section-1-1}

CLAUDE.md is the single most important file in your project. It loads at the start of every session.

**What belongs in CLAUDE.md:**
- Build commands (`bun run dev`, `npm test`)
- Code style rules (naming conventions, import order)
- Architecture decisions (which patterns to use/avoid)
- Hard rules that must always apply ("never commit to main", "always add error handling")
- Anti-patterns Claude has gotten wrong before

**What does NOT belong:**
- Things Claude already does correctly by default (wastes context)
- Workflows that only run sometimes → these become **Skills**
- Style preferences → these become **Hooks**

**Best Practices:**
- Keep it **50–100 lines** max. Long CLAUDE.md = ignored CLAUDE.md
- Ask "Would removing this cause Claude to make mistakes?" — if no, cut it
- Use `@import` to pull in detailed sub-docs (keeps root file lean)
- Store in git and treat it as a first-class team asset
- For rules that are safety-critical ("never push to prod"), use **Hooks** instead — CLAUDE.md compliance is ~70%; Hooks are 100%
- Add to CLAUDE.md after every mistake: "Anytime Claude does something wrong, add it so Claude won't do it next time" — Boris Cherny

**Template:**
```markdown
# Project: [Name]

## Commands
- dev: `bun run dev`
- test: `bun run test -- -t "test name"`
- typecheck: `bun run typecheck`
- lint: `bun run lint`

## Architecture
- State: Zustand (not Redux)
- API: tRPC (not REST controllers)
- DB: Drizzle ORM (never raw SQL in components)

## Rules
- Never modify files in src/legacy/
- Always handle errors with structured logging
- Prefix commits: feat:, fix:, chore:, docs:

## Anti-patterns (learned)
- Do not use `any` TypeScript type
- Do not add console.log — use logger.debug()
- When compacting, always preserve: modified files list, test results, current plan
```

---

### 1.2 — Planning Layer: Plan Mode {#section-1-2}

**The Boris Rule:** "If my goal is to write a Pull Request, I use Plan Mode, go back and forth with Claude until I like its plan. From there, I switch into auto-accept edits mode and Claude can usually 1-shot it. A good plan is really important!"

**How to Use Plan Mode:**
- Trigger: `Shift+Tab` twice (cycles between Normal → Auto-Accept → Plan)
- Use Plan Mode for: multi-file changes, unfamiliar code, architecture decisions
- Skip Plan Mode for: small clear-scope tasks describable in one sentence

**Planning Workflow:**
```
1. Start in Plan Mode
2. Describe goal + constraints
3. Iterate back and forth until plan is right
4. Switch to auto-accept
5. Claude 1-shots the implementation
```

**The Annotated Plan Pattern (Boris Tane variation):**
1. Ask Claude to write `plan.md`
2. Open in editor → add inline notes / challenge assumptions
3. Send back: "Update the plan based on my notes"
4. Repeat until satisfied
5. Say: "Implement it all" — every decision is already made, execution is mechanical

**For Project Planning (not just PRs):**
- Use a dedicated planning subagent (read-only, no write tools)
- Have it produce: PRD → Architecture Doc → Task list with dependencies
- Store outputs as markdown files in the project repo
- Reference them in CLAUDE.md for future sessions

---

### 1.3 — Skills Layer: Repeatable Workflows {#section-1-3}

Skills are SKILL.md files — on-demand knowledge loaded only when relevant, keeping context lean.

**Skills vs CLAUDE.md:**

| CLAUDE.md | Skills |
|---|---|
| Always-on every session | Load only when triggered |
| Short, essential rules | Rich detailed workflows |
| For universal project rules | For specific repeatable tasks |
| ~100 lines max | Can be long and detailed |

**Creating a Skill:**
```markdown
# .claude/skills/api-development/SKILL.md
---
name: api-development
description: Implement REST/tRPC API endpoints following team conventions.
  Use when creating new endpoints, handlers, or API routes.
user-invocable: true
argument-hint: "[endpoint name] [HTTP method]"
---

## API Conventions
- All endpoints must validate input with Zod schemas
- Use structured error responses: { error: string, code: string }
- Every handler needs try/catch with logger.error()
- Rate limiting on all public endpoints

## Steps
1. Create Zod schema in src/schemas/
2. Create handler in src/handlers/
3. Register route in src/router.ts
4. Write unit test in src/handlers/__tests__/
5. Update API docs in docs/api.md
```

**Essential Skills to Build:**
- `api-development` — endpoint conventions
- `frontend-component` — component patterns, design system usage
- `database-migration` — migration steps, rollback procedures
- `code-review` — what to check before merging
- `deployment` — deploy steps, environment checks
- `debugging` — systematic debugging process
- `security-review` — vulnerability checklist
- `testing` — test patterns, coverage requirements

**Slash Commands (inner-loop automation):**
Store in `.claude/commands/`. Commands pre-compute shell output before Claude runs.

```markdown
# .claude/commands/commit-push-pr.md
---
description: Commit, push, and open a PR
---
Current git status:
$(git status --short)

Current diff:
$(git diff --staged)

Create a commit with a conventional commit message, push to origin,
and open a PR. Use the diff to write the PR description.
```

**Boris's Core Commands:**
- `/commit-push-pr` — daily workhorse
- `/fix-issue` — takes issue number, implements fix end-to-end
- `/code-review` — structured review checklist
- `/simplify` — clean up after implementation
- `/deploy` — pre-flight checks + deploy

---

### 1.4 — Subagents Layer: Context-Safe Workers {#section-1-4}

Subagents run in **isolated context windows** and report back summaries. This is the core pattern for preserving context quality.

**The Golden Rule:** A fresh session starts at ~20k tokens. Quality degrades at 20–40% context fill. **Do not let context exceed 60%.** Subagents prevent context rot.

**Mental Model:**
- **Skills** = Knowledge (what Claude knows)
- **Subagents** = Workers (isolated task execution)
- **Agent Teams** = Workers that talk to each other (cross-session)

**When to use subagents:**
- Researching a large codebase (reading 50+ files)
- Security review (read-only, isolated)
- Code simplification (post-implementation cleanup)
- End-to-end verification / testing
- Any task where you want "side effects without pollution"

**Creating Subagents:**
```markdown
# .claude/agents/security-reviewer.md
---
name: security-reviewer
description: Reviews code for security vulnerabilities before deployment.
  Use when: reviewing PRs, before shipping new endpoints, auditing auth flows.
tools: Read, Grep, Glob
model: opus
memory: none
---

You are a senior security engineer. Review code for:
- SQL/XSS/command injection vulnerabilities
- Authentication and authorization flaws
- Secrets or credentials in code
- Insecure dependencies

Return: PASS / WARN / FAIL with specific line references and remediation steps.
```

**Core Subagents to Build:**

| Subagent | Tools | Purpose |
|---|---|---|
| `explore` | Read, Grep, Glob | Codebase research (built-in) |
| `plan` | Read-only | Pre-implementation analysis (built-in) |
| `code-simplifier` | Read, Edit | Post-implementation cleanup |
| `verify-app` | Browser, Bash | E2E testing, UI verification |
| `security-reviewer` | Read, Grep | Pre-deploy security audit |
| `test-writer` | Read, Edit, Bash | Generate + run tests |
| `documentation` | Read, Edit | Docs generation |

**How to invoke:**
```
"Use subagents to investigate how our auth system handles token refresh"
"Use a subagent to review this code for edge cases and security issues"
```

**Agent Teams (parallel execution):**
Enable with `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`.
Use for: refactoring multiple services simultaneously, parallel feature development.
```
"Create an agent team to refactor the auth module:
  one agent handles the User model, another the Session controller,
  third updates the specs."
```

---

### 1.5 — Evaluator Layer: Verification Loop {#section-1-5}

The evaluator closes the quality loop. Without it, Claude verifies using its own judgment (which degrades with context).

**Boris's Evaluator Stack:**
1. **verify-app subagent** — opens browser, tests UI end-to-end, iterates until it works
2. **PostToolUse hook** — formats code after every edit (handles last 10% CI would catch)
3. **Automated tests** — Claude runs tests as part of every PR
4. **code-simplifier subagent** — cleans up architecture after implementation

**PostToolUse Hook (code formatter):**
```json
// .claude/settings.json
{
  "hooks": {
    "PostToolUse": [{
      "matcher": "Edit|Write",
      "hooks": [{
        "type": "command",
        "command": "prettier --write $CLAUDE_TOOL_INPUT_PATH 2>/dev/null || true"
      }]
    }]
  }
}
```

**Challenge Claude as Evaluator:**
- `"Grill me on these changes and don't make a PR until I pass your test"`
- `"Prove to me this works"`
- `"Knowing everything you know now, scrap this and implement the elegant solution"`
- `"After a mediocre fix — what's the real root cause?"`

---

### 1.6 — Parallelism: The Fleet Model {#section-1-6}

Boris ships 20–30 PRs/day using this approach.

**Setup:**
```bash
# Create separate git worktrees per task (avoids conflicts)
git worktree add ../project-feature-a feature-a
git worktree add ../project-feature-b feature-b

# Tab 1
cd ../project-feature-a && claude

# Tab 2
cd ../project-feature-b && claude
```

**5-Tab Strategy:**
- Tab 1: Feature implementation
- Tab 2: Running tests / debugging
- Tab 3: Code review / PR
- Tab 4: Investigating bug
- Tab 5: Documentation / secondary task

**Key:** Use iTerm2/Warp system notifications so you know when any Claude needs input. You're the scheduler, not the typist.

---

### 1.7 — Context Management {#section-1-7}

Context is the primary failure mode. Manage it obsessively.

**Rules:**
- `/clear` between unrelated tasks — always
- Compact proactively at **70% context** (not the default 83.5%)
- Never let context exceed 60% before starting implementation
- If you've corrected Claude >2x on the same issue: clear and restart

**Compaction instructions in CLAUDE.md:**
```markdown
When compacting, always preserve:
- The full list of modified files
- Any test commands and their results
- The current implementation plan
- Error messages that haven't been resolved
```

**Better than /compact:** Dump plan to markdown file → `/clear` → start fresh reading that file. You control exactly what survives.

**Context budget guide:**
- Fresh session overhead: ~20k tokens
- Stay under 60% of 200k = 120k tokens
- Research via subagents, not main session
- Use `/btw` for quick questions (overlay that never enters history)

---

### 1.8 — Hooks: Enforcement at 100% {#section-1-8}

CLAUDE.md rules are followed ~70% of the time. Hooks are 100%.

**Hook types:**
- `PreToolUse` — block actions before they happen
- `PostToolUse` — validate/fix after actions
- `UserPromptSubmit` — intercept before Claude sees prompt
- `Stop` — run after task completion
- `SubagentStart/Stop` — subagent lifecycle

**Use hooks for:**
- Code formatting (PostToolUse on Edit/Write)
- Preventing commits to main (PreToolUse on bash git push)
- Running tests after file changes
- Sound notifications when tasks complete
- Linting on every save

---

### 1.9 — MCP Servers: Extending Capabilities {#section-1-9}

Connect Claude to external systems via MCP. Load only what you need — "If you're using more than 20k tokens of MCPs, you're crippling Claude."

**Essential MCPs for product development:**

| MCP | Use case |
|---|---|
| Playwright | Browser E2E testing, UI verification |
| PostgreSQL/SQLite | Direct schema queries, data inspection |
| GitHub | Issues, PRs, CI status |
| Figma | Design-to-code (read design specs) |
| Slack | Read bug reports, thread context |
| Google Workspace | Sheets, Docs, Calendar for project planning |

**Install:**
```bash
superclaude mcp --servers playwright github postgres
```

---

### 1.10 — Full Project Pipeline {#section-1-10}

<pre class="ascii-diagram">
PLANNING PHASE
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
1. Start in Plan Mode
2. Describe feature/product goal
3. Planning subagent generates: PRD → Architecture → Task list
4. Review + annotate the plan
5. Approve

DEVELOPMENT PHASE
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
6. Switch to auto-accept
7. Claude implements (1-shots with a good plan)
8. Skills activate automatically for domain tasks
9. Subagents handle research/investigation

VERIFICATION PHASE
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
10. code-simplifier subagent cleans up
11. verify-app subagent runs E2E tests
12. security-reviewer subagent checks before shipping
13. PostToolUse hook fixes formatting

SHIP PHASE
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
14. /commit-push-pr slash command
15. @.claude tag on PR → update CLAUDE.md with learnings
16. Living memory grows, quality compounds
</pre>

---

## Part 2: VS CODE COPILOT AGENTS {#part-2-vscode}

The VS Code Copilot agent system mirrors Claude Code's architecture, with some meaningful differences.

### 2.1 — Memory Layer: copilot-instructions.md / AGENTS.md {#section-2-1}

The equivalent of CLAUDE.md in VS Code Copilot:

**Setup:**
```bash
# Type in chat to generate tailored instructions:
/init
```

This analyzes your codebase and auto-generates `/.github/copilot-instructions.md`.

**Hierarchy:**
- `/.github/copilot-instructions.md` — always-on for all requests
- `AGENTS.md` — also supported as always-on
- `CLAUDE.md` — also discovered (cross-compatible!)
- `*.instructions.md` files — file-type specific rules

**File-based instructions example:**
```markdown
# .github/instructions/typescript.instructions.md
---
applyTo: "**/*.ts,**/*.tsx"
---
- Always use strict TypeScript (no `any`)
- Prefer `type` over `interface` for object shapes
- Use Zod for runtime validation
```

---

### 2.2 — Planning: The Plan Agent {#section-2-2}

VS Code has a dedicated **Plan agent** built in:

```
In Chat view → Agents dropdown → Select "Plan"

Prompt: "Create a plan to add OAuth2 authentication
to the app with Google and GitHub providers."
```

The Plan agent:
- Does NOT write code (read-only constraint)
- Generates a structured markdown implementation plan
- Supports handoffs to implementation agent

**Custom Plan Agent (.github/agents/planner.agent.md):**
```markdown
---
description: Generate detailed implementation plans for features or refactors
name: Planner
tools: ['web/fetch', 'search/codebase', 'search/usages']
model: ['Claude Opus 4.6']
handoffs:
  - label: Implement Plan
    agent: agent
    prompt: Implement the plan outlined above.
    send: false
---

You are in planning mode. Generate an implementation plan.
Do NOT make any code changes.

Plan sections:
1. **Overview**: Feature/task description
2. **Requirements**: Functional and non-functional
3. **Architecture**: Components affected, new ones needed
4. **Task list**: Ordered steps with dependencies
5. **Risks**: Edge cases, security considerations
6. **Verification**: How to test this is complete
```

---

### 2.3 — Skills Layer: Agent Skills (SKILL.md) {#section-2-3}

VS Code Agent Skills use the **same SKILL.md format as Claude Code** — fully cross-compatible.

**Location:** `.github/skills/`

**VS Code skill discovery:**
- Auto-triggers based on semantic matching of your request
- Manually invoked via `/skill-name` in chat
- Progressive loading (instructions first, then resources)

**Create a skill:**
```bash
# In VS Code chat:
/create-skill
# Describe what skill to create, it generates the SKILL.md
```

**Example skill:**
```markdown
# .github/skills/webapp-testing/SKILL.md
---
name: webapp-testing
description: Guide for testing web apps with Playwright.
  Use when creating or running browser-based tests.
user-invocable: true
argument-hint: "[page or feature to test]"
---

## When to use
- Creating new Playwright tests
- Debugging failing browser tests
- Setting up test infrastructure

## Testing Standards
1. Every new page needs smoke tests
2. Use Page Object pattern for reusable selectors
3. Tests must run headlessly in CI
4. Screenshots on failure

## Steps
1. Create page object: tests/pages/[name].page.ts
2. Write test: tests/[feature].spec.ts
3. Run: npx playwright test [file]
4. Update CI: .github/workflows/test.yml
```

**Community skills:** `github/awesome-copilot` repo — growing collection of community skills.

---

### 2.4 — Custom Agents: Specialized Roles {#section-2-4}

Custom agents in VS Code = persistent personas with specific tools, models, and handoffs.

**Location:** `.github/agents/`

**Key fields:**
```markdown
---
name: Security Reviewer
description: Reviews code for security vulnerabilities
tools: ['code_search', 'readfile']
model: 'Claude Opus 4.6'
handoffs:
  - label: Fix Issues
    agent: agent
    prompt: Fix the security issues identified in the review.
---
```

**Agent types in VS Code:**

| Type | Where it runs | Best for |
|---|---|---|
| Local | VS Code editor | Interactive, immediate feedback |
| Copilot CLI | Background on machine | Delegated tasks, git worktrees |
| Cloud | GitHub Actions | PRs, async team collaboration |
| Third-party | Anthropic/OpenAI | Claude Code integration |

**Core custom agents to build:**
```
.github/agents/
├── planner.agent.md        # Plan-only, no code changes
├── code-reviewer.agent.md  # Review PRs against standards
├── security-auditor.agent.md # Security scan before deploy
├── architect.agent.md      # Architecture decisions, ADRs
├── tester.agent.md         # Test generation and running
└── documenter.agent.md     # Docs, changelogs, READMEs
```

**Agent handoff chain:**
```
Planner → [handoff] → Implementer → [handoff] → Reviewer → [handoff] → Tester
```

---

### 2.5 — Cloud Agent: Async Background Work {#section-2-5}

The GitHub cloud agent runs on GitHub Actions infrastructure — fire and forget.

**Trigger methods:**
- Assign a GitHub issue to `@copilot`
- Type `/delegate` in Copilot CLI chat
- Comment on PR mentioning `@copilot`
- TODO code actions — hover a `// TODO:` comment → "Delegate to Coding Agent"

**Best for:**
- Long-running implementations you don't want to babysit
- PR reviews while you work on something else
- Dependency updates, migrations, boilerplate
- Async team collaboration

**Workflow:**
```
1. Create GitHub issue with clear spec
2. Assign to @copilot
3. Agent creates PR with empty commit (expected)
4. Pushes incremental commits as it works
5. You review the PR like any other
```

---

### 2.6 — Hooks in VS Code {#section-2-6}

VS Code Copilot supports hooks — scripts triggered at agent lifecycle events.

```json
// .github/copilot-hooks.json
{
  "hooks": {
    "onFileEdit": "prettier --write $FILE",
    "onTaskComplete": "npm test",
    "beforeCommit": "npm run lint && npm run typecheck"
  }
}
```

---

## Part 3: CROSS-PLATFORM FRAMEWORK {#part-3-cross-platform}

This framework works across **both** Claude Code and VS Code Copilot Agents.

### 3.1 — The Universal Project Structure {#section-3-1}

```
your-project/
├── .claude/                    # Claude Code specific
│   ├── agents/                 # Subagents
│   ├── commands/               # Slash commands
│   ├── skills/                 # Skills (also reads .github/skills/)
│   ├── hooks/                  # Hooks
│   └── settings.json           # Permissions, model config
│
├── .github/                    # VS Code Copilot specific
│   ├── copilot-instructions.md # Always-on instructions (≈CLAUDE.md)
│   ├── agents/                 # Custom agents
│   ├── skills/                 # Skills (cross-compatible SKILL.md)
│   ├── instructions/           # File-type specific instructions
│   └── prompts/                # Reusable prompt files
│
├── CLAUDE.md                   # Discovered by both tools
└── AGENTS.md                   # Discovered by VS Code Copilot
```

**Cross-compatibility note:** Both tools now discover and use `CLAUDE.md`, `AGENTS.md`, and SKILL.md files. Skills written in SKILL.md format work across Claude Code, VS Code Copilot, Cursor, and Gemini CLI.

---

### 3.2 — Planning → Development → Ship Pipeline {#section-3-2}

**Phase 1: PROJECT SETUP (once)**
```
Claude Code:  /init or ask "create a CLAUDE.md for this project"
VS Code:      /init in chat → generates copilot-instructions.md
Both:         Build your 5 core skills (api, testing, frontend, db, security)
              Build your 5 core subagents/custom agents
              Set up hooks for formatting, linting
```

**Phase 2: FEATURE PLANNING (per feature)**
```
Claude Code:  Shift+Tab → Plan Mode → iterate plan → approve
VS Code:      Select Plan agent → describe feature → review plan → handoff
Both:         Store plan as plan.md in project
              Annotate plan with your architectural decisions
              Don't let Claude write code until plan is approved
```

**Phase 3: IMPLEMENTATION (per task)**
```
Claude Code:  Switch to auto-accept → Claude 1-shots from plan
VS Code:      Select local agent → implement from plan
Both:         Keep context under 60%
              Use subagents/background agents for research
              Skills auto-activate for domain tasks
```

**Phase 4: VERIFICATION (per PR)**
```
Claude Code:  code-simplifier → verify-app → security-reviewer
VS Code:      Delegate to code-reviewer agent → delegate to tester
Both:         Automated tests must pass
              Format via PostToolUse hooks
              Human reviews the diff
```

**Phase 5: SHIP**
```
Claude Code:  /commit-push-pr
VS Code:      Cloud agent creates PR / local agent creates PR
Both:         Update CLAUDE.md / copilot-instructions.md with learnings
              Tag PR for async review if needed
```

---

### 3.3 — Project Planning (Non-Code Tasks) {#section-3-3}

AI agents aren't just for code. Use them for the full product development lifecycle:

**PRD Generation:**
```
"You are a product manager. Based on this idea: [idea]
Generate a PRD with: problem statement, user stories,
success metrics, scope, and 3-sprint breakdown."
```

**Architecture Decision Records (ADR):**
```
"We need to choose between [option A] and [option B] for [problem].
Generate an ADR with: context, decision drivers, options considered,
decision, and consequences."
```

**Sprint Planning:**
```
"Given this PRD and our velocity of X points/sprint,
break down the work into tasks with: story points, dependencies,
acceptance criteria, and sprint assignment."
```

**Useful planning agents to build:**
- `pm-agent` — PRD generation, user story writing
- `architect-agent` — System design, ADRs, tech decisions
- `scrum-agent` — Sprint planning, task breakdown, estimation
- `qa-agent` — Test plan generation, edge case identification

---

### 3.4 — Anti-Patterns (What Not to Do) {#section-3-4}

| Anti-Pattern | Why It Fails | Fix |
|---|---|---|
| CLAUDE.md > 200 lines | Critical rules get lost in noise | Ruthlessly prune, move workflows to Skills |
| Vibe coding (no plan) | 40 changes you didn't want | Always plan first, especially for multi-file |
| Infinite exploration | Fills context before any code | Scope narrowly, or use subagents |
| Over-engineering with MCPs | >20k tokens of MCPs = 20k for work | Install only what you actively use |
| Too many complex slash commands | Creates anti-pattern; defeats the purpose | Type naturally, build commands for only true inner-loop repeats |
| Long unbroken sessions | Context rot kills output quality | /clear between tasks, fresh session per task |
| No tests or verification | "Plausible-looking" code ships broken | Always provide verification mechanism |
| No hooks for critical rules | 70% CLAUDE.md compliance on safety rules | Move non-negotiable rules to hooks |
| Stale CLAUDE.md | AI learns wrong patterns | Update after every mistake, version-control it |
| Using Sonnet when Opus helps | More steering = slower overall | Opus steers less, often faster end-to-end |

---

### 3.5 — Model Selection Strategy {#section-3-5}

| Task | Recommended Model | Why |
|---|---|---|
| Planning, architecture | Opus (with thinking) | Deep reasoning, fewer steering corrections |
| Feature implementation | Sonnet | Good balance of speed and quality |
| Code search, file exploration | Haiku | Fast, cheap, read-only tasks |
| Security review | Opus | High-stakes, needs careful reasoning |
| Boilerplate generation | Sonnet | Speed matters more |
| Debugging complex issues | Opus (with thinking) | Root cause analysis needs depth |

**Boris's rule:** "Even though Opus is bigger and slower, since you have to steer it less and it's better at tool use, it is almost always faster than using a smaller model in the end."

Route subagents to the right model to control costs:
- Explore subagent → Haiku
- Plan subagent → Sonnet
- Security/Architecture → Opus

---

## Quick Reference {#quick-reference}

### Claude Code Keyboard Shortcuts
- `Shift+Tab` — cycle Plan → Normal → Auto-Accept modes
- `/clear` — fresh context
- `/btw` — quick question, doesn't enter history
- `/compact` — compress context (lossy, use sparingly)
- `/agents` — browse and create subagents
- `/model` — switch model mid-session
- `&` — hand off local session to browser

### VS Code Copilot Quick Start
- `/init` — generate copilot-instructions.md
- `/create-skill` — AI-assisted skill creation
- `/create-agent` — AI-assisted agent creation
- <kbd>Ctrl+Alt+I</kbd> (Mac: <kbd>Ctrl+Cmd+I</kbd>) — open Chat view
- Agents dropdown — switch between local/CLI/cloud/custom
- `@agentname` — invoke a specific custom agent

### Daily Workflow Checklist
```
□ Fresh context for each task (/clear)
□ Plan Mode before any multi-file change
□ Subagents for research, not main session
□ code-simplifier after implementation
□ verify-app before PR
□ /commit-push-pr to ship
□ Update CLAUDE.md if Claude made a mistake
□ Check context % — compact if over 60%
```

---

*Sources: Anthropic Claude Code docs (April 2026), Boris Cherny workflow (Jan 2026),
VS Code Copilot docs (April 2026), community research from builder.io, morphllm.com, levelup.gitconnected.com*
