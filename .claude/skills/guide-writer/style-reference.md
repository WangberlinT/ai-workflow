# GUIDE.md Style Reference

Extracted conventions from the existing guide. Use this as the authoritative
style sheet when writing or editing any chapter.

---

## Document Structure

### Chapter Heading
```
## Chapter N: Title — The Subtitle Layer
```
- Always `##` level
- Format: `Chapter N: Noun — The Adjective Layer`
- Followed by 2–3 paragraphs of conceptual framing (no code yet)
- First paragraph: define the concept in one sentence, then explain the gap it fills
- Second paragraph: scope it relative to other layers

### Section Heading
```
### N.M — Section Title
```
- Always `###` level
- Numbered within the chapter: `1.1`, `1.2`, ..., `1.13`
- Em dash (` — `) separates number from title
- Each section separated by a `---` horizontal rule

### Mandatory Closing Sections (every chapter)

Second-to-last section — **"What We Built"**:
- Shows the current project directory tree (cumulative)
- Bullet list summarizing what was achieved
- Section number: `N.(last-1)`

Last section — **"Evaluation: What X Alone Cannot Do"**:
- Table with columns: Gap | Example | What Solves It
- Each row references a future chapter
- Bridges to the next chapter with a 1–2 sentence transition
- Section number: `N.(last)`

### Chapter Checkpoint
Every chapter ends with a blockquote:
```
> **Chapter N checkpoint:** You should now be able to [list of concrete abilities].
> Try [practical exercise] before moving on.
```

---

## Voice and Tone

- **Second person plural**: "we will build", "you now have", "let us create"
- **Authoritative but approachable**: like an experienced colleague explaining, not lecturing
- **No hedging**: say "this works" not "this might work" or "this could potentially work"
- **No emojis** ever
- **Contractions**: use "do not" over "don't" (formal but not stiff)
- **Active voice**: "Claude reads the frontmatter" not "the frontmatter is read by Claude"
- **Present tense** for how things work: "Skills inject knowledge into context"
- **Future tense** for what we will build: "We will create a review skill"
- **Concrete over abstract**: always follow a concept with a code example within 2–3 paragraphs

---

## Formatting Conventions

### Code Blocks
- Always include a language tag: ` ```yaml `, ` ```markdown `, ` ```bash `, ` ``` ` (plain for directory trees)
- Full, runnable examples — never truncated with `...` unless showing a pattern
- YAML frontmatter examples include the `---` delimiters
- Directory trees use `├──`, `└──`, `│` box-drawing characters with inline `# comments`

### Tables
- Left-aligned headers using `:------` syntax
- Bold the first column when it contains key terms
- Use for comparisons, reference lists, and gap analysis

### Inline Formatting
- **Bold** for key terms on first introduction
- `backticks` for code, file paths, commands, field names
- *Italics* sparingly, mainly in the introductory table ("*What should Claude do?*")
- Em dash ` — ` (surrounded by spaces) for parenthetical clauses

### Emphasis Patterns
- **Bold labels in lists**: `- **Correctness** — Does the code do what it claims?`
- **Blockquotes** for tips, notes, and legacy warnings:
  ```
  > **Note to reader:** ...
  > **Legacy note:** ...
  ```

---

## Content Patterns

### Concept Introduction
1. Define it in one sentence
2. Explain the problem it solves (what happens without it)
3. Show a minimal code example
4. Explain the example line by line

### "Pattern:" Subsections
Named practical patterns appear as `#### Pattern: Name` under a "Practical Patterns" section.
Each pattern:
- Has a one-line description of when to use it
- Shows a complete, copy-pasteable code example
- Includes inline comments explaining non-obvious parts

### Progressive Disclosure
- Early sections cover the concept and minimal usage
- Middle sections cover configuration and mechanics
- Later sections show real-world patterns and composition
- Closing sections evaluate limitations and bridge to next chapter

### Cross-Chapter References
- Forward references: "We will cover this in depth in Chapter N (Topic)"
- Back references: "As we saw in Chapter N, ..."
- Always include chapter number AND topic name

---

## Technical Accuracy Rules

- Every code example must be syntactically valid
- File paths must be consistent across the guide (use the canonical tree from Introduction)
- Frontmatter field names must match the actual Claude Code runtime
- Shell commands must work on macOS (the author's platform)
- When showing a `.claude/` directory tree, it must be cumulative (include everything built so far)

---

## Chapter Template

```markdown
---

## Chapter N: Title — The Subtitle Layer

[2–3 paragraphs: define concept, explain the gap, scope relative to other layers]

---

### N.1 — [Foundation concept]

[Define the basic unit. Show anatomy/structure. Minimal example.]

---

### N.2 — [Where / How it is configured]

[Configuration locations, file formats, scope rules.]

---

### N.3 — [Detailed reference for all options]

[Comprehensive field-by-field or option-by-option reference with examples.]

---

### N.4–N.M — [Progressive sections: mechanics, patterns, real examples]

[Build complexity gradually. Each section should be self-contained but
build on previous ones.]

---

### N.(M+1) — Building Our [Concrete Thing]

[Hands-on walkthrough: build something real for the ai-workflow project.]

---

### N.(M+2) — Practical Patterns

[#### Pattern: Name — reusable recipes]

---

### N.(M+3) — Debugging [Topic]

[Common problems and how to diagnose them.]

---

### N.(last-1) — What We Built

[Cumulative directory tree + bullet summary.]

---

### N.(last) — Evaluation: What [Topic] Alone Cannot Do

[Gap table → bridge to next chapter.]

---

> **Chapter N checkpoint:** ...
```
