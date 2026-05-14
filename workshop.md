---
layout: workshop
title: Agentic Coding in VS Code
permalink: /workshop/
description: A structured workshop for engineers and PMs on VS Code Copilot agent customization — instructions, prompts, skills, agents, subagents, hooks, and MCP.
---

<div class="workshop-hero">
  <p class="page-label">Workshop</p>
  <h1>Agentic Coding in VS Code</h1>
  <p class="hero-sub">VS Code Copilot's agent mode has a deep customization surface — instructions, prompts, skills, agents, subagents, hooks, MCP — but the docs are scattered and the concepts don't obviously connect. This workshop builds the mental model from scratch, walks through two real projects, and ends with hands-on group work. No vibe-coding: every concept earns its place.</p>
</div>

<div class="callout callout-tip">
<p><strong>Companion reference:</strong> The full <a href="/workshop/reference/">Agentic Coding Best Practices doc</a> is embedded at the end of this workshop — a comprehensive cross-platform reference covering Claude Code and VS Code Copilot together, including anti-patterns, model selection, and the complete project pipeline.</p>
</div>

## Who this is for

This workshop is designed for three types of people who keep running into the same wall with VS Code Copilot's agent mode:

- Software engineers who use VS Code Copilot for inline suggestions or basic chat but have never customized an agent, written a skill, or set up a hook
- PMs and non-coders who want to run structured agentic workflows without writing code from scratch — and who keep hearing "just vibe-code it" as the answer
- Anyone who has read the VS Code agent documentation and found the concepts hard to connect into a working mental model

## What you'll have by the end

Three concrete deliverables — one per layer of the reliability stack:

1. A working `.github/copilot-instructions.md` that constrains the agent's behavior for a real project context — the foundation every other asset builds on
2. A reusable `.prompt.md` file that captures a repeatable workflow as a named `/command` — the first step up the reusability ladder
3. A `SKILL.md` bundled with a script and a template — a multi-step capability that loads on demand and can be moved across projects

<p class="section-label">Pages</p>

<div class="workshop-cards">
  <a class="workshop-card" href="/workshop/setup/">
    <p class="workshop-card-num">01</p>
    <h3>Setup</h3>
    <p>Account requirements, extensions, agent mode toggle, MCP scaffolding. Five checks before the first prompt.</p>
  </a>
  <a class="workshop-card" href="/workshop/concepts/">
    <p class="workshop-card-num">02</p>
    <h3>Concepts</h3>
    <p>The full customization surface — all 9 file types explained: where they live, what they do, when to use each. Mapped to the 5-layer reliability stack.</p>
  </a>
  <a class="workshop-card" href="/workshop/flow/">
    <p class="workshop-card-num">03</p>
    <h3>Connecting the Dots</h3>
    <p>Decision tree, reusability ladder, lifecycle pipeline, and 10 anti-patterns with fixes.</p>
  </a>
  <a class="workshop-card" href="/workshop/project-sprint-planning/">
    <p class="workshop-card-num">04</p>
    <h3>Project: Sprint Planning</h3>
    <p>Build a sprint-planning agentic workflow end-to-end — no code required. Instructions → prompt → skill → agent → hook.</p>
  </a>
  <a class="workshop-card" href="/workshop/project-eda-analyser/">
    <p class="workshop-card-num">05</p>
    <h3>Project: EDA Analyser</h3>
    <p>Build a Python EDA CLI with guardrails — instructions, skills, a critic subagent, DuckDB MCP, and a formatting hook.</p>
  </a>
  <a class="workshop-card" href="/workshop/reference/">
    <p class="workshop-card-num">Ref</p>
    <h3>Best Practices Reference</h3>
    <p>Full companion document. Cross-platform framework, anti-patterns, model selection, quick-reference cheatsheet.</p>
  </a>
</div>
