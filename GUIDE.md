# Working with the Claude Code AI Agent

### A Practitioner's Guide to Skills, Commands, Subagents, Plugins, and Hooks

---

## Introduction

Claude Code is not just a coding assistant — it is a programmable AI agent runtime.
Out of the box it reads files, writes code, and runs shell commands. But its real
power emerges when you treat it as an **extensible system** with five interlocking
layers of customization:

```
  ┌─────────────────────────────────────────────────────────────────┐
  │                        Your Workflow                            │
  │                                                                 │
  │  ┌──────────┐  ┌──────────┐  ┌───────────┐  ┌──────┐  ┌─────┐ │
  │  │  Skills   │  │ Commands │  │ Subagents │  │ MCP  │  │Hooks│ │
  │  │          │  │          │  │           │  │Plugins│  │     │ │
  │  │ WHAT to  │  │ HOW to   │  │ WHERE to  │  │ WITH │  │WHEN │ │
  │  │ do       │  │ invoke   │  │ execute   │  │ WHAT │  │ to  │ │
  │  │          │  │          │  │           │  │      │  │react│ │
  │  └────┬─────┘  └────┬─────┘  └─────┬─────┘  └──┬───┘  └──┬──┘ │
  │       │             │              │            │         │    │
  │       └─────────────┴──────┬───────┴────────────┴─────────┘    │
  │                            │                                    │
  │                     Claude Code Runtime                         │
  └─────────────────────────────────────────────────────────────────┘
```

Each layer answers a different question:

| Layer        | Question it answers                                          |
|:-------------|:-------------------------------------------------------------|
| **Skills**   | *What should Claude do?* — domain instructions, templates, and reference material packaged as reusable Markdown files. |
| **Commands** | *How do I invoke it?* — slash commands (`/review`, `/deploy`) give you and Claude named entry points into skills. |
| **Subagents**| *Where does it execute?* — isolated agent contexts with their own tools, models, and permission boundaries. |
| **MCP/Plugins** | *With what external capabilities?* — databases, APIs, design tools, project management, and any service exposed via the Model Context Protocol. |
| **Hooks**    | *When should automation react?* — event-driven shell commands, LLM prompts, or agent invocations that fire at specific lifecycle points. |

Most users never go beyond the first layer. They type natural language, get code
back, and call it a day. This guide is for the practitioners who want more — who
want Claude Code to behave like a **well-configured teammate** rather than a
generic chatbot.

---

### Why This Matters

Consider the difference between these two workflows:

**Without customization:**
```
You:    "Review this PR for security issues"
Claude: (reads files one by one, gives generic advice, misses your
         team's specific security patterns, forgets your coding standards)
```

**With a tuned agent:**
```
You:    /security-review
Claude: (loads your OWASP checklist skill, spawns an Explore subagent to
         scan the full diff, checks against your .eslintrc via MCP,
         auto-formats findings into your team's template, and a PostToolUse
         hook runs your SAST scanner on every file it touches)
```

The second workflow is **deterministic, repeatable, and shareable**. It lives in
version-controlled files that any team member can use. It runs the same way at
2 AM as it does during a pair-programming session.

---

### The Five Layers at a Glance

#### 1. Skills — The Knowledge Layer

Skills are Markdown files (`SKILL.md`) that inject domain-specific instructions
into Claude's context. They can be invoked manually as slash commands or loaded
automatically when Claude determines they are relevant.

```
~/.claude/skills/review/SKILL.md          # personal, all projects
.claude/skills/security-review/SKILL.md   # project, shared via git
```

Key capabilities:
- **YAML frontmatter** controls invocation, model selection, and tool permissions
- **`$ARGUMENTS` substitution** passes user input into the skill template
- **`` !`command` `` dynamic injection** runs shell commands at load time (e.g., `!`git diff``)
- **`context: fork`** runs the skill in an isolated subagent

#### 2. Commands — The Invocation Layer

Every skill with `user-invocable: true` (the default) becomes a `/slash-command`.
Built-in commands (`/compact`, `/clear`, `/plan`, `/hooks`) handle system operations.
MCP servers can expose prompts that appear as `/mcp__server__prompt` commands.

The mental model: **commands are the UI; skills are the logic**.

#### 3. Subagents — The Execution Layer

Subagents are isolated AI processes with their own context window, tool access,
and model selection. They prevent context pollution, enforce least-privilege, and
enable parallelism.

```
.claude/agents/code-reviewer.md    # project-level custom agent
~/.claude/agents/researcher.md     # personal agent, all projects
```

Key architectural properties:
- Each subagent has an **independent context window** (main conversation stays clean)
- **Tool restrictions** enforce least-privilege (e.g., read-only agents)
- **Model routing** sends cheap tasks to Haiku, complex ones to Opus
- Subagents **cannot spawn other subagents** (no recursive nesting)
- **Background execution** allows parallel work streams

#### 4. MCP Servers & Plugins — The Integration Layer

MCP (Model Context Protocol) servers expose external tools to Claude through a
standardized protocol. They turn Claude from a code-only assistant into an agent
that can interact with databases, APIs, design tools, project boards, and any
service you can write a server for.

```bash
claude mcp add --transport http notion https://mcp.notion.com/mcp
claude mcp add --transport stdio postgres -- npx pg-mcp-server
```

Plugins bundle skills, agents, hooks, and MCP servers into distributable packages:

```
my-plugin/
  .claude-plugin/plugin.json    # manifest
  skills/                       # packaged skills
  agents/                       # packaged agents
  hooks/hooks.json              # event handlers
  .mcp.json                     # MCP server configs
```

#### 5. Hooks — The Automation Layer

Hooks are event-driven handlers that fire at specific points in Claude's lifecycle.
They can validate, block, transform, or augment Claude's behavior without requiring
manual intervention.

```
SessionStart → UserPromptSubmit → PreToolUse → PostToolUse → Stop
                                       ↓
                              (can block, modify,
                               or inject context)
```

Three hook types:
- **Command hooks** — run shell scripts, receive JSON on stdin
- **Prompt hooks** — send a single-turn LLM evaluation
- **Agent hooks** — spawn a subagent with tools for multi-turn verification

---

### How This Guide Is Structured

We will build your AI workflow **incrementally**, one layer at a time. Each
chapter adds a new capability and demonstrates the compounding effect of
combining layers:

```
Chapter 1 ── Skills            → Claude knows WHAT to do
Chapter 2 ── Commands          → You control HOW to invoke it
Chapter 3 ── Subagents         → Work happens WHERE it should
Chapter 4 ── MCP & Plugins     → Claude works WITH your tools
Chapter 5 ── Hooks             → Automation reacts WHEN it should
Chapter 6 ── Composition       → All layers work together as a workflow
```

By the end, you will have a **complete, production-grade AI workflow** that is:

- **Deterministic** — the same input produces the same behavior
- **Composable** — layers combine without conflicts
- **Shareable** — lives in version-controlled files your team can use
- **Observable** — hooks provide logging, metrics, and audit trails
- **Safe** — subagents enforce least-privilege, hooks enforce policy

---

### Prerequisites

- Claude Code CLI installed and authenticated
- A project directory (we will use this one: `ai-workflow/`)
- Basic comfort with Markdown and JSON
- Familiarity with shell scripting (bash)

---

### What We Will Build

Throughout this guide, we will construct a working AI workflow for a realistic
scenario: **a development team that wants Claude to assist with code review,
testing, documentation, and deployment** — with guardrails, automation, and
team-wide consistency.

The final system will look like this:

```
ai-workflow/
├── .claude/
│   ├── settings.json              # hooks, permissions, MCP config
│   ├── skills/
│   │   ├── review/SKILL.md        # code review skill
│   │   ├── test-plan/SKILL.md     # test planning skill
│   │   └── deploy-check/SKILL.md  # pre-deploy verification
│   ├── agents/
│   │   ├── explorer.md            # read-only codebase researcher
│   │   ├── reviewer.md            # code review specialist
│   │   └── test-runner.md         # test execution agent
│   └── hooks/
│       ├── lint-on-edit.sh        # auto-lint after file changes
│       ├── block-force-push.sh    # prevent dangerous git ops
│       └── log-actions.sh         # audit trail for all tool uses
├── .mcp.json                      # shared MCP server configs
└── CLAUDE.md                      # project-level memory & conventions
```

---

### Ready?

Turn the page to Chapter 1, where we start with the foundation: **Skills**.
We will write your first `SKILL.md`, understand frontmatter configuration,
and see how a simple Markdown file can transform Claude from a generic assistant
into a domain expert.

> **Note to reader:** At each chapter's end, we will pause to evaluate what we have
> built, identify performance gaps, and decide what the next layer should address.
> This iterative approach mirrors how real workflows evolve — you do not design
> the perfect system upfront; you build, measure, and improve.

---

## Chapters

| # | Title | File |
|:--|:------|:-----|
| 1 | [Skills — The Knowledge Layer](chapters/chapter1.md) | `chapters/chapter1.md` |
| 2 | Commands — The Invocation Layer | `chapters/chapter2.md` |
| 3 | Subagents — The Execution Layer | `chapters/chapter3.md` |
| 4 | MCP & Plugins — The Integration Layer | `chapters/chapter4.md` |
| 5 | Hooks — The Automation Layer | `chapters/chapter5.md` |
| 6 | Composition — The Full Workflow | `chapters/chapter6.md` |
