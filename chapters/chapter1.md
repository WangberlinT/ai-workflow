## Chapter 1: Skills — The Knowledge Layer

A skill is a Markdown file that tells Claude **what to do** in a specific situation.
Without skills, every interaction starts from zero — Claude has no memory of your
coding standards, your team's review checklist, or your deployment process. With
skills, that knowledge is codified, version-controlled, and applied consistently
every time.

Skills are the foundation layer. They do not require commands, subagents, hooks, or
MCP servers. You can get enormous value from skills alone — and everything else we
build in later chapters will build on top of them.

---

### 1.1 — Anatomy of a Skill

A skill lives in a **directory** containing a required `SKILL.md` file and optional
supporting resources:

```
.claude/skills/
└── review/
    ├── SKILL.md           # Main instructions (required)
    ├── checklist.md       # Supporting reference material
    ├── examples/
    │   └── good-review.md # Example output for Claude to follow
    └── scripts/
        └── get-diff.sh    # Script Claude can execute
```

The `SKILL.md` file has two parts: **YAML frontmatter** (configuration) and
**Markdown body** (instructions).

```yaml
---
name: review
description: >
  Review code for correctness, performance, and style.
  Use when reviewing PRs, diffs, or code changes.
allowed-tools: Read, Grep, Glob, Bash(git diff *)
---

## Code Review Instructions

You are a senior engineer performing a code review.

### What to check
1. **Correctness** — Does the code do what it claims?
2. **Edge cases** — What happens with empty input, nulls, large data?
3. **Performance** — Any O(n²) loops, unnecessary allocations, missing indexes?
4. **Style** — Does it follow the project's conventions?

### Output format
For each issue found, report:
- **File and line**: `path/to/file.ts:42`
- **Severity**: critical | warning | nit
- **What**: Description of the issue
- **Why**: Why it matters
- **Fix**: Suggested change
```

That is a complete, functional skill. When invoked, Claude receives these
instructions and follows them. The frontmatter controls **how** the skill runs;
the body tells Claude **what** to do.

---

### 1.2 — Where Skills Live

Skills can be defined at three levels, each with a different scope:

| Level | Path | Scope |
|:------|:-----|:------|
| **Personal** | `~/.claude/skills/<name>/SKILL.md` | All your projects |
| **Project** | `.claude/skills/<name>/SKILL.md` | This project only (commit to git) |
| **Plugin** | `<plugin>/skills/<name>/SKILL.md` | Where the plugin is enabled |

**Personal skills** are your own toolkit — formatting preferences, research
patterns, shortcuts that match your workflow. They travel with you across every
project.

**Project skills** are shared team knowledge. Commit them to git and every team
member using Claude Code gets the same behavior. This is where the real leverage
is: your review checklist, deployment procedure, and testing strategy become part
of the repository.

```
# Personal: your own writing style
~/.claude/skills/explain/SKILL.md

# Project: team's review process (shared via git)
.claude/skills/review/SKILL.md

# Project: team's deploy procedure
.claude/skills/deploy-check/SKILL.md
```

**Automatic discovery in monorepos:** When you are working in a subdirectory like
`packages/frontend/`, Claude also discovers skills from
`packages/frontend/.claude/skills/`. This lets different packages in a monorepo
have their own specialized skills.

> **Legacy note:** Files at `.claude/commands/<name>.md` still work with the same
> frontmatter format. If a skill and a legacy command share the same name, the
> skill takes precedence. New projects should use the skills directory.

---

### 1.3 — Frontmatter Reference

The frontmatter block between `---` markers configures how your skill behaves.
All fields are optional, but `description` is strongly recommended.

#### Core Fields

```yaml
---
name: deploy-check
description: >
  Verify the project is ready for deployment. Run pre-deploy
  checks including test suite, lint, type checking, and
  environment variable validation. Use before any deployment.
---
```

**`name`** — The skill's identifier. Becomes the `/slash-command` name. Lowercase
letters, numbers, and hyphens only (max 64 characters). If omitted, defaults to
the directory name.

**`description`** — What the skill does and **when to use it**. This is the most
important field. Claude reads every skill's description to decide whether to
auto-invoke it. Write it like a search query — include the specific phrases a
user would naturally say.

```yaml
# Bad: vague, no trigger phrases
description: Helps with reviews

# Good: specific, keyword-rich, tells Claude WHEN to use it
description: >
  Review code for correctness, performance, security, and style.
  Use when the user asks to review a PR, check code changes,
  look at a diff, or do a code review.
```

#### Visibility and Invocation Control

```yaml
---
name: coding-standards
description: Our team's TypeScript conventions and patterns
disable-model-invocation: true
---
```

**`disable-model-invocation`** — When `true`, only you can invoke this skill via
`/name`. Claude will never auto-invoke it. **Use this for any skill with
side effects** — deployments, commits, sends, deletes. Default: `false`.

**`user-invocable`** — When `false`, the skill is hidden from the `/` menu. Only
Claude can invoke it (based on the description matching the current task). Use
this for background knowledge that should be loaded automatically but does not
make sense as a user command. Default: `true`.

| Setting | User can invoke | Claude can invoke |
|:--------|:----------------|:------------------|
| *(default)* | Yes | Yes |
| `disable-model-invocation: true` | Yes | No |
| `user-invocable: false` | No | Yes |

**`argument-hint`** — Shown during autocomplete to indicate expected arguments.
Purely cosmetic but helpful for discoverability.

```yaml
argument-hint: "[filename] [format]"
```

#### Tool Permissions

```yaml
---
allowed-tools: Read, Grep, Glob, Bash(npm test *), Bash(git diff *)
---
```

**`allowed-tools`** — Comma-separated list of tools Claude can use **without
asking permission** while the skill is active. Your normal permission settings
still govern all other tools.

You can use glob patterns for Bash commands:

```yaml
# Allow all npm and git commands
allowed-tools: Bash(npm *), Bash(git *)

# Allow only specific operations
allowed-tools: Read, Grep, Bash(npm test), Bash(npm run lint)
```

This is important for workflow fluency. Without `allowed-tools`, Claude stops
and asks permission for every file read and shell command, which destroys the
flow of a multi-step skill. With it, the skill executes smoothly.

#### Execution Context

```yaml
---
context: fork
agent: Explore
model: haiku
---
```

**`context`** — Set to `fork` to run the skill in an isolated subagent. The
subagent gets its own context window, executes independently, and returns a
summary to your main conversation. We will cover this in depth in Chapter 3
(Subagents), but the key insight now: `fork` keeps your main context clean.

**`agent`** — Which subagent type to use when `context: fork` is set. Options:
`Explore`, `Plan`, `general-purpose`, or the name of any custom agent from
`.claude/agents/`. Default: `general-purpose`.

**`model`** — Model to use. Accepts aliases: `opus`, `sonnet`, `haiku`, `inherit`.
Default: `inherit` (same as the main conversation).

#### Skill Metadata

```yaml
---
mode: true
version: "1.0.0"
---
```

**`mode`** — When `true`, the skill appears in a "Mode Commands" section. Use for
skills that establish an operational context (e.g., `debug-mode`, `expert-mode`)
rather than performing a discrete task.

**`version`** — Tracking metadata. Not used by the runtime, but useful for
team communication.

---

### 1.4 — Dynamic Content: Arguments and Shell Injection

Static skills are powerful, but skills that **adapt to context** are
transformative. Two mechanisms make this possible.

#### `$ARGUMENTS` — User Input Substitution

When a user invokes `/review src/auth.ts`, the text `src/auth.ts` is available
as `$ARGUMENTS` inside the skill body:

```yaml
---
name: review
description: Review a specific file
---

Review the file `$ARGUMENTS` for:
1. Correctness
2. Security issues
3. Performance concerns
```

**Positional arguments** are also supported:

| Variable | Value for `/migrate SearchBar React Vue` |
|:---------|:-----------------------------------------|
| `$ARGUMENTS` | `SearchBar React Vue` |
| `$0` | `SearchBar` |
| `$1` | `React` |
| `$2` | `Vue` |

```yaml
---
name: migrate
description: Migrate a component between frameworks
argument-hint: "[component] [from-framework] [to-framework]"
---

Migrate the `$0` component from $1 to $2.
Preserve all existing behavior and tests.
```

If `$ARGUMENTS` does not appear anywhere in the skill body, Claude Code
automatically appends `ARGUMENTS: <user input>` at the end — so arguments are
always available, even if you do not explicitly reference them.

#### `` !`command` `` — Shell Injection at Load Time

The `` !`command` `` syntax runs a shell command **before** the skill content is
sent to Claude. Claude never sees the command — only the output.

```yaml
---
name: pr-review
description: Review the current pull request
context: fork
agent: Explore
---

## Context

Here is the PR diff:

!`git diff main...HEAD`

Here are the changed files:

!`git diff main...HEAD --name-only`

## Your Task

Review this pull request for:
1. Correctness
2. Security issues
3. Missing test coverage
```

When `/pr-review` is invoked:

1. `git diff main...HEAD` runs immediately
2. Its output replaces the `` !`git diff main...HEAD` `` placeholder
3. `git diff main...HEAD --name-only` runs immediately
4. Its output replaces the second placeholder
5. Claude receives the fully-rendered prompt with actual diff content

This is **preprocessing**, not tool use. The commands execute before Claude's
turn begins, so there is no permission prompt and no tool-call overhead. This
makes shell injection ideal for gathering context that every invocation needs.

**Combine both mechanisms** for powerful, parameterized skills:

```yaml
---
name: review-pr
description: Review a specific pull request by number
argument-hint: "[pr-number]"
---

## PR #$0

!`gh pr view $0 --json title,body,additions,deletions`

## Diff

!`gh pr diff $0`

## Review this PR for:
1. Correctness and edge cases
2. Security vulnerabilities
3. Performance regressions
4. Test coverage gaps
```

Now `/review-pr 42` fetches PR #42's metadata and diff, injects them into the
prompt, and Claude reviews with full context from the first token.

---

### 1.5 — Skill Types: A Mental Model

Not all skills serve the same purpose. Recognizing three categories helps you
design the right skill for each situation:

#### Type 1: Reference Skills (Guidelines and Standards)

These inject knowledge into Claude's context. They run inline (no fork) and have
no side effects. Claude applies them alongside whatever task it is performing.

```yaml
---
name: coding-standards
description: >
  TypeScript coding standards and conventions for this project.
  Use when writing or reviewing TypeScript code.
user-invocable: false
---

## TypeScript Standards

- Use `interface` over `type` for object shapes
- Prefer `const` assertions for literal types
- Error handling: always use custom error classes from `src/errors/`
- Naming: PascalCase for types, camelCase for functions, UPPER_SNAKE for constants
- Imports: group by external, internal, types — separated by blank lines
```

Setting `user-invocable: false` means this skill does not clutter the `/` menu.
Claude loads it automatically whenever someone is writing TypeScript in this
project.

#### Type 2: Task Skills (Do Something Specific)

These perform a discrete action. They often use `disable-model-invocation: true`
because they have side effects or cost resources.

```yaml
---
name: test-plan
description: Generate a test plan for a feature or module
disable-model-invocation: true
allowed-tools: Read, Grep, Glob
argument-hint: "[feature-or-module]"
---

Generate a comprehensive test plan for: $ARGUMENTS

## Process
1. Read the implementation files
2. Identify all public functions and their contracts
3. List edge cases for each function
4. Generate a test plan in this format:

### Test Plan: [Feature Name]

| # | Test Case | Input | Expected | Priority |
|---|-----------|-------|----------|----------|
| 1 | ...       | ...   | ...      | P0       |
```

#### Type 3: Mode Skills (Operational Contexts)

These change **how** Claude operates for the rest of the conversation. They
establish rules, personas, or constraints.

```yaml
---
name: debug-mode
description: Enter debugging mode with verbose reasoning
mode: true
---

You are now in DEBUG MODE. For every action you take:

1. **State your hypothesis** before investigating
2. **Show your reasoning** at each step — what you checked and why
3. **List what you ruled out** and why
4. **Propose a fix** only after confirming the root cause
5. **Verify the fix** by explaining why it addresses the root cause

Do not jump to solutions. Follow the evidence.
```

---

### 1.6 — Supporting Files: Skills as Directories

The directory structure is what makes skills more powerful than a single Markdown
file. Supporting files let you separate concerns:

```
.claude/skills/security-review/
├── SKILL.md              # Core instructions (concise, < 500 lines)
├── owasp-checklist.md    # Detailed reference Claude can read
├── examples/
│   ├── xss-finding.md    # Example of a well-written finding
│   └── sqli-finding.md   # Another example
└── scripts/
    └── check-deps.sh     # Script to audit dependencies
```

In `SKILL.md`, reference these files explicitly:

```markdown
## Reference Material

For the complete OWASP checklist, see [owasp-checklist.md](owasp-checklist.md).

For examples of well-written findings, see the [examples/](examples/) directory.

To audit dependencies, run `./scripts/check-deps.sh` in the skill directory.
```

**Why this matters:** Keeping `SKILL.md` under 500 lines ensures it loads quickly
and does not consume excessive context. Detailed reference material lives in
supporting files that Claude reads only when needed.

---

### 1.7 — How Claude Discovers and Invokes Skills

Understanding the runtime behavior helps you write better skills.

**At session start:**
1. Claude Code scans all skill directories (personal, project, plugin)
2. It reads the **frontmatter only** (name and description) from each `SKILL.md`
3. These descriptions are loaded into context as an available skills list
4. The full skill body is **not** loaded yet — this is lazy loading

**When you type a message:**
1. Claude evaluates your message against all skill descriptions
2. If a skill's description matches, Claude invokes it (loads the full body)
3. Shell injections (`` !`command` ``) execute
4. `$ARGUMENTS` are substituted
5. The rendered skill content enters Claude's context
6. Claude follows the instructions

**When you type `/skill-name`:**
1. The skill is invoked directly (no description matching needed)
2. Steps 3–6 above proceed as normal

**Context budget:** Skill descriptions consume approximately 2% of the context
window (fallback: 16,000 characters). If you have many skills, some descriptions
may be excluded from the available skills list. Run `/context` to check which
skills are loaded. You can override the budget with the
`SLASH_COMMAND_TOOL_CHAR_BUDGET` environment variable.

---

### 1.8 — Building Our First Project Skill

Let us build a real skill for our `ai-workflow` project. We will create a
**code review skill** that follows a structured process.

**Step 1: Create the directory structure**

```
.claude/skills/
└── review/
    ├── SKILL.md
    └── checklist.md
```

**Step 2: Write the checklist** (`checklist.md`)

```markdown
# Code Review Checklist

## Correctness
- [ ] Logic matches the stated intent
- [ ] Edge cases handled (null, empty, boundary values)
- [ ] Error paths return meaningful messages
- [ ] No unreachable code

## Security
- [ ] No hardcoded secrets or credentials
- [ ] User input is validated and sanitized
- [ ] SQL queries use parameterized statements
- [ ] File paths are not constructed from user input

## Performance
- [ ] No O(n²) or worse in hot paths
- [ ] Database queries are indexed
- [ ] No unnecessary allocations in loops
- [ ] Large data sets are paginated or streamed

## Style
- [ ] Follows project naming conventions
- [ ] Functions are < 50 lines
- [ ] No commented-out code
- [ ] Imports are organized

## Tests
- [ ] New code has corresponding tests
- [ ] Edge cases are tested
- [ ] Test names describe the scenario, not the implementation
```

**Step 3: Write the skill** (`SKILL.md`)

```yaml
---
name: review
description: >
  Review code for correctness, security, performance, and style.
  Use when the user asks to review code, check a PR, look at changes,
  or do a code review. Works on files, diffs, or staged changes.
allowed-tools: Read, Grep, Glob, Bash(git diff *), Bash(git log *)
argument-hint: "[file-or-path]"
---

## Code Review

You are performing a structured code review.

### What to review

If arguments were provided, review those specific files or paths.
If no arguments were provided, review the current staged changes:

!`git diff --cached --name-only 2>/dev/null || echo "(no staged changes — review the most recently modified files instead)"`

### Process

1. **Read the code** — understand the full context before judging
2. **Check against the checklist** — see [checklist.md](checklist.md)
3. **Prioritize** — focus on correctness and security before style
4. **Be specific** — reference exact lines, not vague areas

### Output Format

For each finding:

**[severity] file.ts:line — Short title**
> Description of the issue, why it matters, and a suggested fix.

Severity levels:
- **critical** — Bug, security hole, or data loss risk. Must fix.
- **warning** — Potential problem or significant improvement. Should fix.
- **nit** — Style, naming, or minor improvement. Nice to fix.

### Summary

End with:
- Total findings by severity
- Overall assessment: APPROVE, REQUEST CHANGES, or NEEDS DISCUSSION
- One sentence on the strongest aspect of the code
```

This skill demonstrates several patterns:
- **Rich description** with trigger phrases for auto-invocation
- **`allowed-tools`** for fluent execution without permission prompts
- **Shell injection** to auto-detect staged changes
- **Supporting file** reference for the checklist
- **Structured output** format for consistent, readable reviews

---

### 1.9 — A Second Skill: Test Planning

Let us build one more to see how skills complement each other:

```yaml
---
name: test-plan
description: >
  Generate a test plan for a module, feature, or function.
  Use when the user wants to plan tests, needs test cases,
  or asks what to test.
disable-model-invocation: true
allowed-tools: Read, Grep, Glob
argument-hint: "[module-or-feature]"
---

## Test Plan Generation

Generate a comprehensive test plan for: $ARGUMENTS

### Process

1. **Find the implementation** — locate all relevant source files
2. **Identify the public API** — list every exported function/class/method
3. **Map the dependencies** — what does this code call? What calls it?
4. **Enumerate test cases** — for each public function:
   - Happy path (normal input, expected output)
   - Edge cases (empty, null, boundary values, max size)
   - Error cases (invalid input, network failures, timeouts)
   - Integration points (does it interact with DB, filesystem, APIs?)

### Output Format

### Test Plan: [Module Name]

**Files analyzed:** (list of source files read)

| # | Function | Test Case | Input | Expected Output | Priority |
|---|----------|-----------|-------|-----------------|----------|
| 1 | ...      | ...       | ...   | ...             | P0       |

**Coverage notes:**
- Untestable areas and why
- Suggested mocking strategy
- Integration test recommendations

**Estimated effort:** (rough count of test cases by priority)
```

Note that `disable-model-invocation: true` prevents Claude from auto-invoking
this skill. Generating a test plan is an intentional action — you invoke it
explicitly with `/test-plan auth-module`.

---

### 1.10 — Practical Patterns

Here are patterns that work well in production:

#### Pattern: Git-Aware Skills

Inject git context so Claude knows what has changed:

```yaml
---
name: changelog
description: Generate a changelog entry from recent commits
---

## Recent Commits

!`git log --oneline -20`

## Changed Files Since Last Tag

!`git diff $(git describe --tags --abbrev=0 2>/dev/null || echo HEAD~20)..HEAD --stat`

## Task

Write a changelog entry covering these changes. Group by:
- Features
- Bug fixes
- Breaking changes
```

#### Pattern: Environment-Aware Skills

Adapt behavior based on the project's toolchain:

```yaml
---
name: ci-fix
description: Diagnose and fix CI failures
allowed-tools: Read, Grep, Glob, Bash(npm *), Bash(yarn *), Bash(pnpm *)
---

## Project Context

Package manager: !`[ -f pnpm-lock.yaml ] && echo pnpm || ([ -f yarn.lock ] && echo yarn || echo npm)`
Node version: !`node --version 2>/dev/null || echo "not installed"`
Test framework: !`grep -l "jest\|vitest\|mocha" package.json 2>/dev/null | head -1 && cat package.json | grep -o '"jest"\|"vitest"\|"mocha"' | head -1 || echo "unknown"`

## Task

Diagnose the CI failure described in $ARGUMENTS and fix it.
```

#### Pattern: Template-Driven Output

For skills that produce structured documents, keep the template in a supporting
file:

```
.claude/skills/adr/
├── SKILL.md
└── template.md
```

```yaml
# SKILL.md
---
name: adr
description: Create an Architecture Decision Record
disable-model-invocation: true
argument-hint: "[decision-title]"
---

Create an Architecture Decision Record for: $ARGUMENTS

Follow the template in [template.md](template.md) exactly.
Fill in every section. Use concrete details, not placeholders.
```

```markdown
# template.md
# ADR-NNN: [Title]

## Status
Proposed | Accepted | Deprecated | Superseded

## Context
What is the issue that we're seeing that motivates this decision?

## Decision
What is the change that we're proposing and/or doing?

## Consequences
What becomes easier or harder because of this change?
```

---

### 1.11 — Debugging Skills

When a skill does not behave as expected:

**Skill not auto-invoked?**
- Check the description. Is it specific enough? Does it contain the phrases
  you naturally use?
- Run `/context` to verify the skill is loaded (it may have been excluded due
  to context budget limits)
- Check `disable-model-invocation` — if `true`, Claude cannot auto-invoke it

**Skill invoked but behavior is wrong?**
- Read the rendered output. Shell injections may have failed silently
- Check that supporting file paths are correct (relative to `SKILL.md`)
- Verify `allowed-tools` includes everything the skill needs

**Arguments not working?**
- Ensure `$ARGUMENTS` or `$0`, `$1`, etc. appear in the body
- If they do not appear, arguments are appended at the end (which may not be
  where you need them)

**Context too long?**
- Keep `SKILL.md` under 500 lines
- Move detailed reference to supporting files
- Use `context: fork` for skills that produce large outputs

---

### 1.12 — What We Built

At the end of this chapter, our project structure looks like this:

```
ai-workflow/
├── .claude/
│   └── skills/
│       ├── review/
│       │   ├── SKILL.md        # Code review with structured output
│       │   └── checklist.md    # Detailed review checklist
│       └── test-plan/
│           └── SKILL.md        # Test plan generation
└── GUIDE.md                    # This guide
```

We now have two skills that:
- Give Claude **specific, structured instructions** instead of ad-hoc prompts
- Are **version-controlled** and shareable with the team via git
- Use **shell injection** to gather context automatically
- Use **`allowed-tools`** for smooth execution
- Produce **consistent, formatted output** every time

---

### 1.13 — Evaluation: What Skills Alone Cannot Do

Skills are powerful, but they have limitations that motivate the remaining layers:

| Gap | Example | What Solves It |
|:----|:--------|:---------------|
| **No named entry point** | You have to remember skill names or hope Claude auto-invokes them | **Commands** (Ch. 2) — explicit `/slash-commands` with discoverability |
| **Shared context window** | A review skill's large diff output crowds out your conversation | **Subagents** (Ch. 3) — `context: fork` for isolation |
| **No external tools** | Cannot query your database, check Jira, or read Figma designs | **MCP/Plugins** (Ch. 4) — external tool integration |
| **No automation** | You must manually invoke skills; nothing happens automatically on events | **Hooks** (Ch. 5) — event-driven triggers |
| **No guardrails** | A skill can suggest a force-push; nothing stops Claude from executing it | **Hooks** (Ch. 5) — PreToolUse validation |

The next chapter addresses the first gap: making skills **discoverable and
invocable** through the command system. We will see how skills and commands form
a natural pair — skills are the logic, commands are the interface.

---

> **Chapter 1 checkpoint:** You should now be able to create a `SKILL.md` with
> frontmatter, use `$ARGUMENTS` for parameterization, inject runtime context
> with `` !`command` ``, and organize supporting files in the skill directory.
> Try building a skill for a real workflow in your project before moving on.
