# Guide Progress Tracker

## Chapters

| # | Title | File | Status |
|:--|:------|:-----|:-------|
| Intro | Introduction | `GUIDE.md` | Done |
| 1 | Skills — The Knowledge Layer | `chapters/chapter1.md` | Done |
| 2 | Commands — The Invocation Layer | `chapters/chapter2.md` | Done |
| 3 | Subagents — The Execution Layer | `chapters/chapter3.md` | Not started |
| 4 | MCP & Plugins — The Integration Layer | `chapters/chapter4.md` | Not started |
| 5 | Hooks — The Automation Layer | `chapters/chapter5.md` | Not started |
| 6 | Composition — The Full Workflow | `chapters/chapter6.md` | Not started |

## What Comes Next

Chapter 3: Subagents. Covers isolated agent contexts with their own context
windows, tool restrictions, and model routing. Links to `context: fork` from
skills and shows how to build custom agents for parallel, least-privilege
execution.

## File Structure

```
ai-workflow/
├── GUIDE.md                          # Introduction + table of contents
├── PROGRESS.md                       # This file
├── chapters/
│   ├── chapter1.md                   # Skills
│   ├── chapter2.md                   # Commands
│   ├── chapter3.md                   # Subagents (next)
│   ├── chapter4.md                   # MCP & Plugins
│   ├── chapter5.md                   # Hooks
│   └── chapter6.md                   # Composition
└── .claude/
    └── skills/
        └── guide-writer/
            ├── SKILL.md              # Writing/editing skill
            └── style-reference.md    # Style conventions
```
