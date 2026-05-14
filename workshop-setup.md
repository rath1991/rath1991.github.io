---
layout: workshop
title: Setup — Agentic Coding in VS Code
permalink: /workshop/setup/
description: Account requirements, VS Code version, extension installation, agent mode activation, MCP scaffolding, and a five-point sanity checklist before the first prompt.
---

# Setup

Five things to verify before the first agent prompt. Each one is a silent failure mode if skipped.

---

## 1. Account & Subscription

GitHub Copilot requires a GitHub account with an active subscription. Agent mode is available on all tiers, but premium request limits vary.

| Plan | Inline suggestions | Chat | Agent mode | Notes |
|---|---|---|---|---|
| **Copilot Free** | ✓ (limited) | ✓ (limited monthly) | ✓ (limited) | Monthly caps on completions and chat turns |
| **Copilot Pro** | ✓ unlimited | ✓ unlimited | ✓ | Uses premium requests for agent turns |
| **Copilot Pro+** | ✓ unlimited | ✓ unlimited | ✓ more models | Higher premium request allocation |
| **Business / Enterprise** | ✓ | ✓ | ✓ | Organisation-managed, audit logs, policy controls |

Agent mode runs use **premium requests** — each tool call the agent makes counts against your plan's premium request allocation. Free-tier users will hit limits faster during multi-step agent sessions.

<div class="callout callout-warn">
<p>As of April 2026, new sign-ups for Copilot Pro and Pro+ are temporarily paused. Existing subscribers are unaffected. Check <a href="https://code.visualstudio.com/docs/copilot/setup" target="_blank">the official setup page</a> for current availability.</p>
</div>

---

## 2. Install VS Code & Extensions

<div class="step">
  <div class="step-num">1</div>
  <div class="step-body">
    <strong>VS Code 1.99 or later</strong>
    <p>Agent mode requires VS Code 1.99+. Check your version: <kbd>Help</kbd> → <kbd>About</kbd>. If you're on an older build, update before continuing — agent features are silently unavailable on older versions.</p>
  </div>
</div>

<div class="step">
  <div class="step-num">2</div>
  <div class="step-body">
    <strong>Install GitHub Copilot extension</strong>
    <p>Open Extensions (<kbd>Ctrl+Shift+X</kbd> / <kbd>Cmd+Shift+X</kbd>) and search for "GitHub Copilot". Install the extension published by GitHub. This provides inline completions.</p>
  </div>
</div>

<div class="step">
  <div class="step-num">3</div>
  <div class="step-body">
    <strong>Install GitHub Copilot Chat extension</strong>
    <p>Search for "GitHub Copilot Chat" and install the companion extension. This is required for the chat panel, agent mode, and all customization features. Both extensions must be present and up to date — agent mode degrades silently if the Chat extension is stale.</p>
  </div>
</div>

<div class="step">
  <div class="step-num">4</div>
  <div class="step-body">
    <strong>Sign in with GitHub</strong>
    <p>Hover over the Copilot icon in the Status Bar and select <strong>Use AI Features with Copilot…</strong>. Follow the sign-in prompts. Once signed in, the Copilot icon should appear active (not crossed out) in the Status Bar.</p>
  </div>
</div>

<div class="callout callout-warn">
<p>Both extensions must be installed and updated. The Copilot extension alone is not sufficient for agent mode — the Chat extension is what enables the chat panel, agent dropdown, and the full customization layer.</p>
</div>

---

## 3. Enable Agent Mode

Agent mode is not on by default in all VS Code builds. Verify the setting is active.

<div class="step">
  <div class="step-num">1</div>
  <div class="step-body">
    <strong>Open Settings</strong>
    <p>Press <kbd>Ctrl+,</kbd> (Windows / Linux) or <kbd>Cmd+,</kbd> (Mac) to open the Settings editor.</p>
  </div>
</div>

<div class="step">
  <div class="step-num">2</div>
  <div class="step-body">
    <strong>Search for agent mode</strong>
    <p>Type <code>chat.agent.enabled</code> in the search box. Toggle it to <strong>true</strong>. If you don't see the setting, your VS Code build is older than 1.99.</p>
  </div>
</div>

<div class="step">
  <div class="step-num">3</div>
  <div class="step-body">
    <strong>Open the Chat panel</strong>
    <p>Press <kbd>Ctrl+Alt+I</kbd> (Windows / Linux) or <kbd>Ctrl+Cmd+I</kbd> (Mac) to open the Chat view. Look for a mode dropdown near the chat input. It should show options including "Ask", "Edit", and <strong>Agent</strong>.</p>
  </div>
</div>

---

## 4. Initialise Project Instructions

Run `/init` in the Copilot Chat panel. This analyses your codebase and auto-generates `.github/copilot-instructions.md` with conventions derived from your actual code — languages, frameworks, naming patterns.

```bash
# In the Copilot Chat input:
/init
```

Review and trim the generated file. It will be verbose — cut anything that doesn't answer the question "would removing this cause the agent to make a mistake?" Keep it under 100 lines.

The `.github/` folder is where all your customization assets live:

```
your-project/
├── .github/
│   ├── copilot-instructions.md   ← always-on context
│   ├── agents/                   ← custom agents
│   ├── skills/                   ← skill packages
│   ├── prompts/                  ← reusable prompts
│   └── hooks/                    ← lifecycle hooks
└── .vscode/
    └── mcp.json                  ← MCP server connections
```

---

## 5. MCP Scaffolding (Optional)

MCP servers give the agent access to external tools — databases, APIs, services. Configure them in `.vscode/mcp.json`:

```json
{
  "servers": {
    "my-database": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres"],
      "env": {
        "POSTGRES_CONNECTION_STRING": "${env:PG_CONN}"
      }
    }
  }
}
```

MCP tools are only available in Agent mode (not regular chat). Each server's tool schema consumes context — load only what the current session needs.

**Official docs:** [Add MCP servers in VS Code](https://code.visualstudio.com/docs/copilot/customization/mcp-servers)

---

## Sanity Checklist

Run through this before the first prompt:

- [ ] VS Code version ≥ 1.99 (`Help → About`)
- [ ] GitHub Copilot extension installed and signed in (Status Bar icon is active)
- [ ] GitHub Copilot Chat extension installed and up to date
- [ ] `chat.agent.enabled` set to `true` in Settings
- [ ] Chat panel open, mode dropdown shows "Agent" as an option

<div class="callout callout-tip">
<p>Once you see the Agent mode option in the dropdown, you're ready. The rest of this workshop builds on this foundation — every file type you'll create goes into <code>.github/</code> or <code>.vscode/</code>.</p>
</div>
