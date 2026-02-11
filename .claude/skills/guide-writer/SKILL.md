---
name: guide-writer
description: >
  Write, edit, and improve chapters of the Claude Code guide.
  Use when the user wants to write a new chapter, edit content,
  improve writing, check consistency, or review the tutorial.
disable-model-invocation: true
allowed-tools: Read, Grep, Glob, Bash(wc *)
argument-hint: "[action] [chapter-or-section]"
---

## Guide Writer

You are editing the practitioner's guide "Working with the Claude Code AI Agent."

### File Layout

```
GUIDE.md              — Introduction only (do NOT put chapters here)
PROGRESS.md           — Chapter status tracker
chapters/chapter1.md  — Skills (done)
chapters/chapter2.md  — Commands
chapters/chapter3.md  — Subagents
chapters/chapter4.md  — MCP & Plugins
chapters/chapter5.md  — Hooks
chapters/chapter6.md  — Composition
```

### What to Read (and What NOT to Read)

**Always read first:**
1. [style-reference.md](style-reference.md) — formatting, voice, structure rules
2. `PROGRESS.md` — what is done, what comes next

**Read only when needed:**
- The specific `chapters/chapterN.md` you are writing or editing
- The **last 2 sections** of the previous chapter (for voice calibration and
  continuity), not the whole chapter
- `GUIDE.md` introduction only if you need the canonical directory tree or
  chapter overview

**Never read all chapters at once.** Each chapter is self-contained. Work on
one file at a time.

### Actions

Parse `$ARGUMENTS` for the action:

- **write N** — Draft chapter N in `chapters/chapterN.md`
- **edit N.M** — Improve section M of chapter N
- **review N** — Audit chapter N for issues (output a report, do not edit)
- **continue** — Check `PROGRESS.md`, confirm next chapter, then write it

### Writing Rules

Follow [style-reference.md](style-reference.md) strictly. Key points:

- Heading: `## Chapter N: Title — The Subtitle Layer`
- Sections: `### N.M — Title`, separated by `---`
- Voice: second person plural, no contractions, no emojis, no hedging
- Every concept gets a code example within 2–3 paragraphs
- End with: "What We Built" (cumulative tree) + "Evaluation" (gap table) + checkpoint blockquote
- Target: 600–900 lines per chapter, 40–80 lines per section

### Accuracy Checks

Before finishing any write or edit:

- [ ] Code examples are syntactically valid and runnable
- [ ] Frontmatter fields match the actual Claude Code runtime
- [ ] File paths match the canonical tree in `GUIDE.md` introduction
- [ ] Section numbering is sequential with no gaps
- [ ] Cross-chapter references use both chapter number and topic name
- [ ] "What We Built" tree is cumulative (includes all previous chapters)
- [ ] Shell commands work on macOS

### After Writing

Update `PROGRESS.md` to mark the chapter as done and set the next target.
