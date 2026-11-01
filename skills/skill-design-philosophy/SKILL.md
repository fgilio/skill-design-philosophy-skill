---
name: skill-design-philosophy
description: >
  Design philosophy for creating outstanding Claude Code skills.
  Use when: creating a new skill, improving an existing skill, designing script interfaces.
user-invocable: true
disable-model-invocation: false
---

# Skill Design Philosophy

Philosophy and principles only. Toolchain-specific quirks and implementation details belong in their respective skill docs (e.g., php-cli-skill-builder).

**Goal**: Create skills that eliminate errors by design. Script skills hide tool quirks behind scripts that handle everything internally. Knowledge skills encode judgment the model can't infer on its own.

**Loading economics**: the description is always in context (it is the routing layer that decides invocation), while the body loads only when the skill is used. So descriptions must be trigger-rich with the key use case first, and bodies carry deferred cost: long reference material and full examples are close to free.

Official docs: <https://code.claude.com/docs/en/skills.md>

---

## Core Principles

### 1. Zero-Error by Design

The skill should make it impossible to fail due to underlying tool quirks. If a CLI needs `--description` not `--body`, the script uses the right flag. If URLs need encoding, the script encodes them. Users never see the complexity.

This includes validating early: check dependencies, inputs, and formats before doing any work. If an external API changed its response shape, the script should fail with a clear message, not silently produce garbage.

```bash
# Bad: User needs to know about URL encoding
api projects/org%2Frepo/merge_requests/123

# Good: Script handles it
skill-mr-view org/repo 123
```

```bash
# Validate before doing anything
command -v tool >/dev/null 2>&1 || { echo '{"error": "tool not installed"}' >&2; exit 1; }
[[ "$PROJECT" =~ ^[^/]+/[^/]+$ ]] || { echo '{"error": "Invalid project format"}' >&2; exit 1; }
```

For scripts that create resources or trigger side effects, consider idempotency. A "create MR" script invoked twice shouldn't create duplicates.

### 2. Intuitive Interfaces

Commands should read naturally and not require memorization. Sensible defaults eliminate unnecessary arguments: target branch defaults to `main`, filters default to the most common case (`open` not `all`).

```bash
# Bad: Requires knowing tool internals
tool mr create --repo org/project --title "Fix bug" --description "Details" --target-branch main --yes

# Good: Reads naturally, obvious what it does
skill-mr-create org/project "Fix bug" --description "Details"
```

**Naming convention**: Prefix scripts with skill name for global access: `{skill-name}-{resource}-{action}` (e.g., `gitlab-mr-create`, `browser-screenshot`). This prevents namespace collisions when scripts are in PATH.

### 3. Predictable Output

Two rules:

- **stdout** is for the consumer (Claude or a pipe). JSON for data operations, plain text for logs/traces.
- **stderr** is for problems. JSON errors with exit 1. Usage help with exit 1.

Never mix formats within a script. Never return nothing. Use an empty JSON array if there are no results.

```bash
# Bad: Inconsistent contract
script1  # returns JSON
script2  # returns "Success!"
script3  # returns nothing

# Good: Consistent contract
script1  # returns JSON
script2  # returns {"status": "created", "id": 42}
script3  # returns [] (empty array)
```

```bash
# Bad error
Error: something went wrong

# Good error (JSON to stderr)
{"error": "Invalid project format. Use: org/project"}
```

### 4. Self-Contained Scripts

Each skill is a self-contained unit. No external package dependencies, no build steps.

Share code within a skill freely (e.g., a `lib.ts` for auth and validation). Never share code between skills, copy patterns instead. Each script owns its entry point with its own usage/help and argument parsing.

### 5. Atemporal Writing

Write skills as timeless reference documentation. Describe the system's current state as if it has always worked this way.

Avoid transitional language ("no longer", "now", "previously", "updated to", "changed from"). Skills get read fresh in future sessions where the transition is irrelevant noise. The reader cares about how things work today, not their history.

Heuristic: if a sentence becomes wrong or confusing the moment someone reads it outside the context of the change that prompted it, rewrite it in the present tense describing only the current behavior.

```markdown
# Bad: anchored to a specific change
The script no longer requires --yes; it now prompts interactively instead.

# Good: timeless
The script prompts interactively before destructive actions.
```

This applies to SKILL.md, reference docs, script comments, and usage strings alike. Put transition notes in commit messages and changelogs, not in the skill itself.

### 6. Precise but Generic

Every concrete detail in a rule or example is either load-bearing or a liability. Precise means the load-bearing specifics stay exact: the boundary the rule pins, the identifier typed verbatim, the command that must not change. Generic means nothing else is specific: no incidental domain vocabulary, no names the lesson does not need. Generic never means vague. A line with no load-bearing specifics teaches nothing and fails the opposite way.

Test each detail: does generalizing it change what the reader does? If yes, it is the precision, keep it exact (a pinned version stays when the rule depends on it). If no, it is a liability that taxes attention, narrows perceived scope, or drifts out of date.

**Build examples from the stock material every tutorial shares: users, posts, tags, orders, emails, invoices.** Skills are read mid-task, so every example competes with whatever domain the agent currently holds (a payment flow, a migration, a UI bug). An example built on foreign concepts forces the reader to absorb vocabulary that serves neither the rule being taught nor the work in progress. Familiar vocabulary carries near-zero novelty, so all attention lands on the lesson.

```php
// Bad: imports crypto vocabulary to teach control flow
if (! HmacKey::isStrong($webhookKey)) {

// Good: same lesson, vocabulary every reader already holds
if (! $user->hasVerifiedEmail()) {
```

Two exceptions:

- Terms that are the subject of their rule stay. A rule about the word "idempotent" must name it.
- The skill's declared domain is free vocabulary. Laravel terms cost nothing in a Laravel skill. Declared means the skill's own subject, not whichever repo the session happens to sit in.

### 7. Descriptions Are Read-Triggers

The description routes. The body teaches. A description that summarizes the body's mechanics loads that vocabulary into every session, which defeats the deferral the body exists for. The same rule governs any resident pointer to deferred content: a CLAUDE.md line referencing a doc carries the trigger (when to read) and the path, never a summary of what the doc contains. This is principle 6 at the routing layer: precise about when to read, silent about what will be learned.

Test: does this phrase help decide WHEN to invoke or read (routing key: keep) or does it describe what the reader WILL LEARN inside (content summary: cut)?

```yaml
# Bad: content summary — the body's mechanics leak into every session
description: Reviews code through three personas across report, address, and comment modes, scoring each file on twelve dimensions.

# Good: trigger + routing keys
description: Audit code against the team's standards in one pass. Use when reviewing a PR, a branch, or pending changes.
```

Two things stay in the description even though they describe behavior: safety properties ("never mutates remote state") and precedence clauses ("OVERRIDES the global skill"). They change the invocation decision itself, which makes them routing. Mode lists belong in `argument-hint`, not the description.

---

## Knowledge Skills

Not every skill wraps a tool. Knowledge skills (style guides, domain references, writing voices) ship judgment instead of scripts. Principles 5 and 6 apply in full, plus:

**Rules state boundaries, examples calibrate them.** The model already holds generic best practices. What it can't infer is where your boundaries sit, and the examples are the only encoding of that. Cutting examples to save tokens regresses every rule toward the model's defaults.

**Teach through contrast pairs.** A DO/DON'T or keep/cut pair pins the decision boundary a prose rule can only gesture at. The keep examples matter most: without them a prohibition gets over-applied and strips legitimate uses.

**Give the core rule a handle.** A compact phrase, either pretrained ("tracer bullet", "idempotent") or coined for the skill ("precise but generic"), packs the whole behavior into a term the reader thinks with. Anchor it twice: in the description, where it routes invocation, and in the body, where it governs execution. The same test as any detail applies: when a paraphrase changes nothing about what the reader does, the handle is decoration.

**Steps that demand evidence get done.** In a procedural skill, a step whose output is unobservable gets skimmed under load. Make each step produce a named artifact: a ratio, a list of surfaces checked, a verdict per item. The artifact is the completion criterion, so a reader cannot report the step done without doing the work.

**An optional step is an unfinished trigger.** "Optionally" hands the decision to the reader's judgment, which varies with capability, so at the floor the step degrades to skipped or inconsistently applied. Every step is either gated by a condition the reader can check or absent. "Re-run the suite when any file under tests/ changed" gates. "Optionally re-run the suite" abstains.

**Practice what the skill preaches.** A style skill that breaks its own rules undermines them. Lint the skill against itself.

**Calibrate at the capability floor.** Docs serve a reader that is weaker and fresher than the author: a lower-capability model in a clean context window. The author is the wrong judge on both axes, because it cannot un-know what it infers for free and cannot un-read what it just wrote. Spawn the reader instead of simulating it: give an independent low-effort agent the block and a task the block governs, never the expected answer, and let its output decide. When the verdict is load-bearing, use three agents with majority vote, and tell each to answer only and never perform the task.

---

## Script Template

The rhythm every script follows. For a full working example, see `~/.claude/skills/browser/scripts/`.

```bash
#!/bin/bash
# [One-line description]
# Usage: skill-resource-action <required> [optional]
# Handles: [what quirks this abstracts away]
set -e

# Check dependencies
# Usage guard
# Parse + validate args
# Set defaults
# Do the work (handle quirks internally)
# Output JSON
```

TypeScript skills follow the same shape but run via `bun run` with no npm dependencies beyond Bun builtins.

### Handling Errors

API calls can fail. Don't let raw errors leak through:

```bash
RESPONSE=$(command "endpoint" 2>&1) || {
  jq -n --arg msg "Request failed: $RESPONSE" '{"error": $msg}' >&2
  exit 1
}

# Validate response is valid JSON before processing
echo "$RESPONSE" | jq -e . >/dev/null 2>&1 || {
  jq -n '{"error": "Invalid response"}' >&2; exit 1
}

echo "$RESPONSE" | jq '{id, name}'
```

### Pagination

Most APIs paginate by default. For complete results, use the tool's pagination flag or set `per_page` explicitly. Be careful with unbounded pagination on large datasets.

---

## Project Structure

### Repo layout

Skills live at `~/dev/skills/<name>-skill/` (the `-skill` suffix keeps repos scannable and unambiguous across an org). A symlink at `~/.claude/skills/<name>` makes the skill visible to Claude Code:

```text
~/dev/skills/gitlab-skill/SKILL.md               # source of truth
~/.claude/skills/gitlab  -> ~/dev/skills/gitlab-skill   # symlink (consumer-facing)
```

The symlink name matches the skill's `name` frontmatter, not the repo name. The consumer-facing handle (`gitlab`) stays terse while the repo carries the `-skill` suffix.

Two variants work:

- **Flat**: SKILL.md at repo root, symlink → repo root. Use for symlink-only repos.
- **Plugin layout**: SKILL.md inside `skills/<name>/`, symlink → `<repo>/skills/<name>`. Use when the repo is also distributed as a Claude Code plugin. Plugin installers discover skills under the `skills/` directory and miss a root-level SKILL.md, so plugin repos need this layout.

### Directory layout

```text
skill-name/
├── SKILL.md              # Required: primary reference for Claude Code and humans
├── SPEC.md               # Optional: collaborative spec (kept after implementation)
├── reference.md          # Optional: deep documentation (or references/ directory)
├── setup.sh              # Optional: install entry point (see Installation)
├── package.json          # TypeScript only: bin field for global access via `bun link`
└── scripts/
    ├── setup.md              # Optional: human installation guide
    ├── lib.ts                # TypeScript only: shared infra within this skill
    └── skill-resource-action # The actual scripts
```

### Installation

Skills with runtime dependencies ship a `setup.sh` at the repo root. Consumers run `./setup.sh` and never need to know whether the skill uses Bun, Composer, pip, or anything else. The script resolves its own location, checks for the runtime, and runs the installer:

```bash
#!/usr/bin/env bash
set -euo pipefail
cd "$(dirname "$0")"

if ! command -v bun >/dev/null 2>&1; then
  echo "error: bun is required. Install from https://bun.sh" >&2
  exit 1
fi

bun install
```

When scripts need global CLI access, extend `setup.sh` to link them after install:

```bash
# TypeScript skills with `bin` entries in package.json:
bun install
bun link         # exposes scripts as ~/.bun/bin/<command>

# Bash skills:
for script in scripts/skill-*; do
  ln -sf "$PWD/$script" ~/.local/bin/$(basename "$script")
done
```

Principle 1 at the install boundary: the consumer runs one command, the script handles quirks. Skills without runtime dependencies (pure markdown, shell scripts that only need a symlink) can skip `setup.sh`.

### SKILL.md frontmatter

```yaml
---
name: skill-name          # display label only; the directory name sets the /command
description: >
  [What it does].
  [When to use it].
  Trigger words: [comma-separated keywords].
user-invocable: true            # false hides it from the / menu (background knowledge)
disable-model-invocation: false # true stops Claude loading it automatically
---
```

The skill listing truncates the description at 1,536 characters.

---

## Checklist for New Skills

### Before Building

- [ ] Identify the underlying tool's quirks and pain points
- [ ] List common operations users need
- [ ] Design intuitive command signatures

### Documentation

- [ ] SKILL.md with clear usage examples
- [ ] Comment in each script explaining what quirks it handles
- [ ] `reference.md` or `references/` directory if raw API/tool docs would be useful

### Testing

- [ ] Happy path works
- [ ] Validation errors return clean JSON
- [ ] Edge cases handled (empty results, special chars)

### For Knowledge Skills

- [ ] Description is trigger-rich with the key use case first
- [ ] Precise but generic: every remaining specific detail changes reader behavior
- [ ] Every rule that needs a boundary has a contrast pair
- [ ] The core rule has a handle that appears in both description and body
- [ ] Every procedural step names the artifact it produces
- [ ] No step is optional: each is gated by a checkable condition or absent
- [ ] Atemporal: no transition language
- [ ] The skill passes its own rules
