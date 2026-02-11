## Chapter 2: Commands — The Invocation Layer

A command is a named entry point that connects you to a skill, a built-in
operation, or an external prompt. Without commands, you rely on Claude to guess
which skill is relevant — or you describe what you want from scratch every time.
With commands, you type `/review` and the right instructions load instantly,
every time.

Chapter 1 gave Claude knowledge through skills. This chapter gives you control
through commands. The mental model is simple: **skills are the logic, commands
are the interface**. Every workflow benefits from having both.

---

### 2.1 — The Command System

Claude Code's command system draws from three sources:

1. **Built-in commands** — system operations hardcoded into the runtime
   (`/compact`, `/clear`, `/model`)
2. **Skill commands** — any skill with `user-invocable: true` becomes a slash
   command automatically
3. **MCP prompt commands** — prompts exposed by MCP servers appear as
   `/mcp__server__prompt`

When you type `/` in the REPL, Claude Code merges all three sources into a
single completion menu. You do not need to remember where a command comes
from — the interface is uniform.

```
claude> /
  Built-in:
    /compact        Compress conversation context
    /clear          Reset conversation history
    /model          Switch AI model
    ...
  Skills:
    /review         Code review with structured output
    /test-plan      Generate test plan from source
    ...
  MCP:
    /mcp__github__create-pr    Create a pull request
    ...
```

The important insight is that **you already know how to create commands**. If
you completed Chapter 1, you have two skills that are already commands. The
`user-invocable` field defaults to `true`, so every skill you build is
immediately available as `/skill-name` in the REPL.

This chapter goes deeper: we will explore the full command system, learn to
design commands with clear argument interfaces, understand how Claude discovers
and selects commands, and build a new command that ties everything together.

---

### 2.2 — Built-in Commands Reference

Claude Code ships with built-in commands for session management, configuration,
diagnostics, and navigation. These are always available regardless of what
skills or MCP servers you have configured.

**Session Management**

| Command | What It Does |
|:--------|:-------------|
| `/compact [instructions]` | Compress conversation history to free context space. Optional instructions focus the compression (e.g., `/compact focus on the auth refactor`). |
| `/clear` | Reset conversation history completely. Starts a fresh context. |
| `/exit` | Exit the REPL session. |

**Configuration**

| Command | What It Does |
|:--------|:-------------|
| `/config` | Open the interactive configuration panel for viewing and modifying settings. |
| `/model` | Switch between available models (Haiku, Sonnet, Opus). |
| `/memory` | Open `CLAUDE.md` and other memory files for direct editing. |
| `/init` | Initialize a new project with a `CLAUDE.md` guide. |
| `/permissions` | Manage tool permissions interactively. |

**Context and Diagnostics**

| Command | What It Does |
|:--------|:-------------|
| `/context` | Visualize current context window usage as a colored grid. |
| `/cost` | Show token usage and cost statistics for the current session. |
| `/doctor` | Run installation health checks and diagnose configuration issues. |
| `/status` | Display current session status. |

**Features**

| Command | What It Does |
|:--------|:-------------|
| `/hooks` | View and manage configured hooks through an interactive menu. |
| `/skills` | List all available skills in the current context. |
| `/agents` | View and manage custom subagents. |

**Mode Switching**

| Command | What It Does |
|:--------|:-------------|
| `/plan` | Toggle Plan Mode for exploratory analysis without making changes. |
| `/vim` | Toggle Vim keybinding mode for the input editor. |

> **Note to reader:** Run `/help` at any time to see the complete list of
> commands available in your session, including any custom skills and MCP
> prompts you have configured.

---

### 2.3 — From Skill to Slash Command

Every skill directory maps to a command through a simple naming convention:

```
.claude/skills/review/SKILL.md         →  /review
.claude/skills/test-plan/SKILL.md      →  /test-plan
.claude/skills/deploy-check/SKILL.md   →  /deploy-check
~/.claude/skills/my-tool/SKILL.md      →  /my-tool
```

The directory name becomes the command name. The `SKILL.md` file provides the
instructions that load when the command is invoked. No additional registration
or configuration is needed.

When a user types `/review`, Claude Code:

1. Locates `.claude/skills/review/SKILL.md` (or the personal equivalent at
   `~/.claude/skills/review/SKILL.md`)
2. Reads the frontmatter for configuration (`allowed-tools`, `model`, `context`)
3. Substitutes any argument variables (`$ARGUMENTS`, `$0`, `$1`, ...)
4. Executes any shell injections (`` !`command` ``)
5. Injects the expanded Markdown body into Claude's context as instructions
6. Applies tool and model overrides from the frontmatter

The result is that Claude receives the full skill content and follows it. From
the user's perspective, one keystroke combination replaces a paragraph of
instructions.

**The description field drives discovery.** When users browse commands with
`/help` or tab completion, the `description` from frontmatter appears alongside
the command name:

```yaml
---
name: review
description: >
  Review code for correctness, performance, and security.
  Uses the team checklist and produces structured output.
---
```

```
claude> /help
  ...
  /review    Review code for correctness, performance, and security.
  ...
```

A good description is specific enough that users know what the command does
without reading the skill file. Write descriptions as imperative sentences that
state the outcome.

---

### 2.4 — Command Arguments

Commands become far more powerful when they accept arguments. Claude Code
provides two substitution mechanisms for passing user input into skill templates.

**`$ARGUMENTS` — The Full Argument String**

The variable `$ARGUMENTS` expands to everything the user typed after the
command name:

```
/review src/auth/login.ts --focus security
        └──────────────────────────────────┘
                     $ARGUMENTS
```

Use `$ARGUMENTS` when your skill needs free-form input — a file path, a
description, or a set of flags:

```yaml
---
name: review
description: Review the specified file or current diff
---

Review the following code: $ARGUMENTS

Use the checklist below to structure your findings.
...
```

**Positional Arguments — `$0`, `$1`, `$2`, ...**

When your command has a structured interface with distinct parameters, use
positional arguments:

```
/migrate-component SearchBar React Vue
                   └──────┘ └────┘ └──┘
                      $0      $1    $2
```

Here is a skill that uses positional arguments to migrate a component between
frameworks:

```yaml
---
name: migrate-component
description: Migrate a named component from one framework to another
user-invocable: true
allowed-tools: Read, Glob, Grep, Edit, Write
---

## Migration Task

Migrate the component **$0** from **$1** to **$2**.

### Steps

1. Find all files related to `$0` using Glob and Grep
2. Read the current $1 implementation
3. Rewrite it using idiomatic $2 patterns
4. Update imports in all consuming files
5. Verify the migration compiles
```

**When Arguments Are Missing**

If a user invokes `/migrate-component` without arguments, the positional
variables remain as literal strings in the expanded text. Claude sees the raw
`$0` tokens and typically asks the user for the missing values. Design your
skill body to handle this gracefully:

```markdown
Migrate the component **$0** from **$1** to **$2**.

If any of the above values appear as literal "$0", "$1", or "$2", ask the
user to provide: component name, source framework, and target framework.
```

**Combining `$ARGUMENTS` with Shell Injection**

Arguments and shell injection compose naturally. A skill can use arguments to
select what context to inject:

```yaml
---
name: review-branch
description: Review all changes on a named branch
---

Review the changes introduced by the branch: $0

Here is the diff:

!`git diff main...$0`
```

When invoked as `/review-branch feature/auth`, the skill first substitutes `$0`
with `feature/auth`, then executes `git diff main...feature/auth` to inject the
diff.

---

### 2.5 — Command Scopes and Discovery

Commands inherit the scope of their underlying skills. Claude Code searches for
skills in three locations, checked in this order:

| Scope | Location | Shared Via | Use Case |
|:------|:---------|:-----------|:---------|
| **Project** | `.claude/skills/` | Git | Team-wide commands for this repository |
| **Personal** | `~/.claude/skills/` | Not shared | Your own commands across all projects |
| **Plugin** | `<plugin>/skills/` | Plugin registry | Third-party command packages |

When two skills share the same directory name, project scope takes precedence
over personal scope. This lets you override a personal command with a
project-specific version.

**Discovering Available Commands**

Claude Code provides several ways to find commands:

**Tab completion** — type `/` and press Tab to see all available commands.
Continue typing to filter:

```
claude> /rev<Tab>
  /review
  /review-branch
```

**The `/help` command** — prints a categorized list of all available commands
with descriptions. This is the most comprehensive view, showing built-in
commands, skill commands, and MCP prompt commands together.

**The `/skills` command** — lists all loaded skills with their metadata. Useful
for verifying that your skill was discovered and its description is correct.

> **Note to reader:** If a skill does not appear in `/help` or tab completion,
> check that the directory structure is correct (`skills/<name>/SKILL.md`), that
> the file parses without YAML errors, and that `user-invocable` is not set to
> `false`.

---

### 2.6 — Controlling Invocation Behavior

Two frontmatter fields control who can invoke a skill and when:

**`user-invocable`** (default: `true`)
- When `true`: the skill appears as a `/slash-command` in the menu
- When `false`: the skill does not appear in the command menu; only Claude can
  invoke it autonomously

**`disable-model-invocation`** (default: `false`)
- When `false`: Claude can invoke the skill autonomously when it determines the
  skill is relevant
- When `true`: only explicit user invocation via `/command` triggers the skill

These two fields are independent, creating four possible combinations:

| `user-invocable` | `disable-model-invocation` | Who Can Invoke | Use Case |
|:-----------------|:---------------------------|:---------------|:---------|
| `true` | `false` | User or Claude | **Default.** General-purpose skills. |
| `true` | `true` | User only | Sensitive operations (deploy, delete) that should never run without explicit intent. |
| `false` | `false` | Claude only | Background skills that inject context when relevant but need no command name. |
| `false` | `true` | Nobody | Effectively disabled. Useful for temporarily shelving a skill. |

**When to restrict invocation:**

Use `disable-model-invocation: true` for commands that have side effects or make
irreversible changes. A deploy-check skill should run only when you explicitly
type `/deploy-check`, not when Claude decides a deployment might be happening:

```yaml
---
name: deploy-check
description: Run pre-deployment verification checklist
user-invocable: true
disable-model-invocation: true
---
```

Use `user-invocable: false` for skills that augment Claude's behavior without
needing a command name. A coding-standards skill that should always apply when
Claude writes code does not need to be a command:

```yaml
---
name: coding-standards
description: >
  Apply team coding standards when writing or reviewing code.
  Enforces naming conventions, error handling patterns, and
  documentation requirements.
user-invocable: false
disable-model-invocation: false
---
```

---

### 2.7 — MCP Prompts as Commands

MCP (Model Context Protocol) servers can expose **prompts** — predefined
instruction templates that appear as slash commands in Claude Code. This extends
your command system to include external tools and services.

When an MCP server is connected, its prompts appear using this naming
convention:

```
/mcp__<server-name>__<prompt-name>
```

For example, if you configure a GitHub MCP server:

```bash
claude mcp add github -- npx @anthropic/github-mcp-server
```

And that server exposes a prompt called `create-pr`, it becomes available as:

```
/mcp__github__create-pr
```

MCP prompts accept arguments just like skill commands:

```
/mcp__github__create-pr "Add auth module" "Implements OAuth2 flow"
```

**MCP prompts vs. skill commands:**

| Aspect | Skill Commands | MCP Prompt Commands |
|:-------|:---------------|:--------------------|
| **Source** | Local `SKILL.md` files | Remote MCP server |
| **Naming** | `/skill-name` | `/mcp__server__prompt` |
| **Customizable** | You edit the Markdown directly | Defined by the server |
| **Offline** | Always available | Requires server connection |
| **Arguments** | `$ARGUMENTS`, `$0`, `$1` | Server-defined parameters |

MCP prompts are most useful for integrations that need external state — creating
issues in a tracker, querying a database, or interacting with a deployment
pipeline. We will explore MCP servers in depth in Chapter 4 (MCP & Plugins).

---

### 2.8 — Building Our Deploy-Check Command

Let us build a practical command for our `ai-workflow` project. A
**deploy-check** command runs a pre-deployment verification checklist — the kind
of command you want to invoke explicitly before every release.

Create the skill directory and file:

```
.claude/skills/deploy-check/SKILL.md
```

```yaml
---
name: deploy-check
description: >
  Run pre-deployment verification checklist. Checks for uncommitted
  changes, failing tests, lint errors, and dependency vulnerabilities.
user-invocable: true
disable-model-invocation: true
allowed-tools: Bash, Read, Glob, Grep
---

## Pre-Deployment Verification

Run the following checks in order. Stop at the first failure and report
the issue clearly. If all checks pass, output a deployment-ready summary.

### 1. Git Status

Verify the working tree is clean:

!`git status --porcelain`

If the output above is not empty, report uncommitted changes and stop.

### 2. Branch Check

Confirm we are on the expected deployment branch: **$0**

If "$0" appears literally (no branch was specified), default to `main`.

Current branch:

!`git branch --show-current`

### 3. Test Suite

Run the project test suite:

```bash
npm test 2>&1 || echo "TESTS FAILED"
```

Report any failures with file names and error messages.

### 4. Lint Check

Run the linter:

```bash
npm run lint 2>&1 || echo "LINT FAILED"
```

### 5. Dependency Audit

Check for known vulnerabilities:

```bash
npm audit --production 2>&1 || echo "AUDIT ISSUES"
```

### Summary Format

After all checks pass, output:

```
## Deploy Check: PASSED

- Branch: [branch name]
- Tests: [pass count] passed
- Lint: clean
- Audit: no vulnerabilities
- Commit: [short SHA]
- Time: [timestamp]
```
```

This command demonstrates several design choices:

- **`disable-model-invocation: true`** prevents deployment checks from running
  unless explicitly requested
- **`allowed-tools: Bash`** grants shell access for running tests and linters
- **Positional argument `$0`** accepts an optional branch name with a sensible
  default
- **Shell injection** injects `git status` and `git branch` output at load time
- **Structured output format** ensures consistent, scannable results every run

Invoke it with a target branch:

```
claude> /deploy-check main
```

Or without arguments to use the default:

```
claude> /deploy-check
```

---

### 2.9 — Designing Command Interfaces

Good commands share characteristics with good CLI tools. Here are principles for
designing commands that your team will actually use.

#### Principle: Name Commands as Verbs

Command names should describe actions, not objects:

| Good | Bad | Why |
|:-----|:----|:----|
| `/review` | `/code-reviewer` | Commands are actions, not identities |
| `/deploy-check` | `/deployment` | Specific action, not a vague noun |
| `/test-plan` | `/tests` | Describes what it produces |
| `/migrate-component` | `/component-migration` | Verb-first reads naturally after `/` |

#### Principle: Keep Arguments Minimal

If a command needs more than three positional arguments, the interface is too
complex. Use `$ARGUMENTS` as free-form text instead, or design the skill to ask
Claude to gather missing information interactively:

```yaml
---
name: scaffold
description: Generate project scaffolding from a description
---

Generate project scaffolding based on the following requirements:

$ARGUMENTS

If the requirements above are unclear or incomplete, ask the user to
clarify before generating any files.
```

#### Principle: Write Descriptions for Scanning

Users see descriptions in `/help` output and tab completion. Front-load the
action:

```yaml
# Good — action first, details second
description: Review code for security vulnerabilities and OWASP compliance

# Bad — buries the action
description: >
  A comprehensive tool that helps developers find potential security issues
```

#### Principle: Provide Defaults for Optional Arguments

When a command accepts optional arguments, always document the default behavior
in the skill body. This prevents Claude from hallucinating values when the user
omits them:

```markdown
Review the file: $0

If "$0" appears literally, review the most recently modified file instead:

!`git diff --name-only HEAD~1 | head -1`
```

---

### 2.10 — Practical Patterns

#### Pattern: Context-Gathering Command

Some commands exist purely to gather and present information. They do not modify
files — they read the codebase and produce a report:

```yaml
---
name: codebase-overview
description: Generate a structured summary of project architecture
user-invocable: true
allowed-tools: Read, Glob, Grep
---

## Codebase Overview

Analyze this project and produce a structured overview.

### Project metadata

!`cat package.json 2>/dev/null || cat Cargo.toml 2>/dev/null || echo "No manifest found"`

### File structure (first 50 entries)

!`find . -type f -not -path './.git/*' -not -path './node_modules/*' | head -50`

### Report format

Produce a report with these sections:
1. **Tech stack** — languages, frameworks, build tools
2. **Directory layout** — purpose of each top-level directory
3. **Entry points** — main files, CLI entry, API routes
4. **Dependencies** — key external libraries and their roles
5. **Test coverage** — test framework, test file locations
```

This pattern is useful for onboarding new team members or giving Claude a fast
orientation on an unfamiliar codebase.

#### Pattern: Guard Command

A guard command validates preconditions before a risky operation. It produces a
go/no-go decision:

```yaml
---
name: pre-merge
description: Validate that the current branch is safe to merge
user-invocable: true
disable-model-invocation: true
allowed-tools: Bash, Read, Grep
---

## Pre-Merge Validation

Check whether the current branch is ready to merge into $0 (default: main).

### Checks

1. No merge conflicts with target branch:
   ```bash
   git merge-tree $(git merge-base HEAD $0) HEAD $0
   ```

2. All CI-relevant files are committed:
   !`git status --porcelain`

3. No TODO or FIXME comments in changed files:
   ```bash
   git diff $0...HEAD --name-only | xargs grep -n "TODO\|FIXME" || echo "None found"
   ```

### Output

If all checks pass:
```
READY TO MERGE into [target branch]
```

If any check fails, explain the issue and suggest a fix.
```

#### Pattern: Templated Generator

A generator command creates files from a template. Arguments control what gets
generated:

```yaml
---
name: new-component
description: Generate a new React component with tests and styles
user-invocable: true
allowed-tools: Write, Read, Glob
---

## Generate Component: $0

Create a new React component named **$0** with the following files:

- `src/components/$0/$0.tsx` — component implementation
- `src/components/$0/$0.test.tsx` — unit tests
- `src/components/$0/$0.module.css` — CSS module

### Style conventions

!`cat .prettierrc 2>/dev/null || echo "No Prettier config"`

### Existing component for reference

!`ls src/components/ 2>/dev/null | head -1 | xargs -I{} head -30 src/components/{}/{}.tsx 2>/dev/null || echo "No existing components"`

Follow the patterns established in existing components. If no existing
components are found, use standard React conventions with TypeScript and
functional components.
```

---

### 2.11 — Debugging Commands

When a command does not behave as expected, work through these checks:

**Command does not appear in `/help` or tab completion?**

- Verify the directory structure: `.claude/skills/<name>/SKILL.md` — the file
  must be named exactly `SKILL.md`
- Check for YAML frontmatter syntax errors. A missing `---` delimiter or
  invalid YAML will prevent the skill from loading
- Confirm `user-invocable` is not set to `false`
- Run `/skills` to see whether the skill was discovered at all

**Command loads but arguments are not substituted?**

- Check that you used `$ARGUMENTS` or `$0`, `$1` (with the dollar sign prefix)
  in the Markdown body
- Verify you typed arguments after the command name: `/review src/auth.ts`,
  not `/review` followed by `src/auth.ts` on a new line
- Remember that arguments are substituted as literal text — they are not parsed
  or validated by Claude Code

**Shell injection fails silently?**

- The `` !`command` `` syntax runs at skill load time, not during Claude's
  execution
- If the command fails, it produces no output — the placeholder is replaced with
  an empty string
- Test the shell command manually in your terminal first
- Ensure the command works from the project root directory (that is where shell
  injections execute)

**Command runs but Claude ignores the instructions?**

- Check whether the skill body is too long. Skills over 500 lines may be
  compressed or partially dropped from context
- Verify that the instructions are clear and imperative. Vague guidance produces
  vague results
- Try adding explicit reinforcement at the end of the skill body: "Do not
  deviate from these instructions"

**Claude invokes the wrong command autonomously?**

- When `disable-model-invocation` is `false`, Claude selects skills based on
  description matching. If two skills have similar descriptions, Claude may
  pick the wrong one
- Make descriptions distinct and specific to each command's purpose
- Set `disable-model-invocation: true` on skills that should only run when
  explicitly requested

---

### 2.12 — What We Built

At the end of this chapter, our project structure looks like this:

```
ai-workflow/
├── .claude/
│   └── skills/
│       ├── review/
│       │   ├── SKILL.md        # Code review with structured output
│       │   └── checklist.md    # Detailed review checklist
│       ├── test-plan/
│       │   └── SKILL.md        # Test plan generation
│       └── deploy-check/
│           └── SKILL.md        # Pre-deployment verification
└── GUIDE.md                    # This guide
```

We now have three commands that demonstrate the full range of the invocation
layer:

- **`/review`** — a general-purpose command that Claude can also invoke
  autonomously when it detects review-related work
- **`/test-plan`** — a parameterized command that accepts arguments for
  targeting specific modules
- **`/deploy-check`** — a guarded command restricted to explicit user
  invocation only, with structured pass/fail output

Each command is:

- **Discoverable** — appears in `/help` and tab completion with a clear
  description
- **Parameterized** — accepts arguments through `$ARGUMENTS` and positional
  variables
- **Scoped** — invocation behavior is controlled through `user-invocable` and
  `disable-model-invocation`
- **Self-documenting** — the `description` field explains what the command does
  without reading the skill file

---

### 2.13 — Evaluation: What Commands Alone Cannot Do

Commands give us named entry points, but they still operate within the
constraints of a single conversation:

| Gap | Example | What Solves It |
|:----|:--------|:---------------|
| **Shared context window** | `/review` on a large diff floods your conversation with thousands of lines | **Subagents** (Ch. 3) — isolated context windows that do not pollute the main conversation |
| **No parallel execution** | Running `/review` and `/test-plan` sequentially wastes time when they could run simultaneously | **Subagents** (Ch. 3) — background execution enables parallel work streams |
| **No external data** | `/deploy-check` cannot query your CI/CD API, check Jira for open blockers, or verify staging health | **MCP & Plugins** (Ch. 4) — external tool integration through standardized protocols |
| **No automatic triggers** | You must remember to run `/deploy-check` before every deployment manually | **Hooks** (Ch. 5) — event-driven automation that fires at the right moment |
| **No permission boundaries** | A command can use any tool Claude has access to, even when it only needs read access | **Subagents** (Ch. 3) — tool restrictions enforce least-privilege execution |

The next chapter addresses the most pressing gap: **context isolation**. When a
review skill processes a 2,000-line diff, that entire diff enters your
conversation history and crowds out everything else. Subagents solve this by
executing work in their own context window and returning only the results.

---

> **Chapter 2 checkpoint:** You should now be able to explain the three sources
> of commands (built-in, skill, MCP), design a skill with `$ARGUMENTS` and
> positional parameters, control invocation behavior with `user-invocable` and
> `disable-model-invocation`, and discover commands using `/help` and tab
> completion. Try adding a `/deploy-check` command to your own project before
> moving on.
