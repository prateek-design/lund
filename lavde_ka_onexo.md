# Lavde Ka Onexo (What it is and what it does)

## What is this?
**Lavde Ka Onexo** is a short, human-friendly documentation file that explains the **Onexo** product in plain language.

It’s meant to help anyone—new team members, reviewers, or stakeholders—understand:
- what Onexo is,
- why it exists,
- what it helps you do,
- and where to start.

## What does Onexo do?
Onexo helps teams build and manage AI-powered workflows by providing a dashboard that can:

### 1) Organize AI building blocks
- Create and manage **skills** (reusable capabilities)
- Create and manage **agents** (orchestrators that use skills)
- Configure **pipelines** (automated multi-step flows)

### 2) Run and monitor work
- Execute workflows and tool calls
- Inspect run history and telemetry
- Debug what happened during an execution (inputs, steps, outcomes)

### 3) Connect real external tools
- Integrate with provider systems such as **GitHub** and **Bitbucket**
- Use configured actions to read or update data (e.g., repositories, PR-related operations)

### 4) Keep changes auditable
- Changes to skills/agents/pipelines are tracked in a registry
- You can review versions and diffs over time

## Why does it matter?
Instead of treating AI as a black box, Onexo makes AI workflow development:
- **structured** (clear components: skills/agents/pipelines)
- **observable** (you can see runs and tool call details)
- **integrated** (connect to the systems your team already uses)
- **iterable** (versioned updates and controlled changes)

## Quick starting point
If you’re new, a good order is:
1. Understand the **Agent** and **Skill** relationship
2. Look at a **Pipeline** that runs an agent
3. Use telemetry to confirm the workflow behaves as expected

## Notes
This repository file is intentionally written as an introductory doc. If you’d like, I can also help expand it with:
- screenshots/links to the dashboard UI
- a concrete example workflow
- a glossary of common terms (skills, agents, pipelines, telemetry)
