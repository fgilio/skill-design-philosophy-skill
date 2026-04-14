---
name: skill-design-philosophy
description: >
  Design philosophy for creating outstanding Claude Code skills.
  Use when: creating a new skill, improving an existing skill, designing script interfaces.
user-invocable: false
disable-model-invocation: false
---

# Skill Design Philosophy

Philosophy and principles only. Toolchain-specific quirks and implementation details belong in their respective skill docs (e.g., [php-cli-builder](../php-cli-builder/SKILL.md)).

**Goal**: Create skills that eliminate errors by design. Users should never encounter tool quirks - scripts handle everything internally.

Official docs: https://code.claude.com/docs/en/skills.md

---

## Core Principles

### 1. Zero-Error by Design

The skill should make it impossible to fail due to underlying tool quirks. If a CLI needs `--description` not `--body`, the script uses the right flag. If URLs need encoding, the script encodes them. Users never see the complexity.

This includes validating early: check dependencies, inputs, and formats before doing any work. If an external API changed its response shape, the script should fail with a clear message - not silently produce garbage.

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

Commands should read naturally and not require memorization. Sensible defaults eliminate unnecessary arguments - target branch defaults to `main`, filters default to the most common case (`open` not `all`).

```bash
# Good: Reads naturally, obvious what it does
skill-mr-create org/project "Fix bug" --description "Details"

# Bad: Requires knowing tool internals
tool mr create --repo org/project --title "Fix bug" --description "Details" --target-branch main --yes
```

**Naming convention**: Prefix scripts with skill name for global access: `{skill-name}-{resource}-{action}` (e.g., `gitlab-mr-create`, `browser-screenshot`). This prevents namespace collisions when scripts are in PATH.

### 3. Predictable Output

Two rules:
- **stdout** is for the consumer (Claude or a pipe). JSON for data operations, plain text for logs/traces.
- **stderr** is for problems. JSON errors with exit 1. Usage help with exit 1.

Never mix formats within a script. Never return nothing - use an empty JSON array if there are no results.

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
# Good error (JSON to stderr)
{"error": "Invalid project format. Use: org/project"}

# Bad error
Error: something went wrong
```

### 4. Self-Contained Scripts

Each skill is a self-contained unit. No external package dependencies, no build steps.

Share code within a skill freely (e.g., a `lib.ts` for auth and validation). Never share code between skills - copy patterns instead. Each script owns its entry point with its own usage/help and argument parsing.

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

---

## Script Template

The rhythm every script follows. For a full working example, see `~/.claude/skills/gitlab/scripts/`.

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

### Directory layout

```
skill-name/
├── SKILL.md              # Required: primary reference for Claude Code and humans
├── SPEC.md               # Optional: collaborative spec (kept after implementation)
├── reference.md          # Optional: deep documentation (or references/ directory)
├── package.json          # TypeScript only: bin field for global access via `bun link`
└── scripts/
    ├── setup.md              # Optional: human installation guide
    ├── lib.ts                # TypeScript only: shared infra within this skill
    └── skill-resource-action # The actual scripts
```

### Global access

Skills with scripts should be installable for global CLI access:

**TypeScript (Bun):** Add `bin` field to `package.json`, then `bun link`. Commands go to `~/.bun/bin/`.

**Bash:** Symlink scripts to `~/.local/bin/`:
```bash
for script in ~/.claude/skills/skill-name/scripts/skill-*; do
  ln -sf "$script" ~/.local/bin/$(basename "$script")
done
```

### SKILL.md frontmatter

```yaml
---
name: skill-name          # lowercase, hyphens only
description: >
  [What it does].
  [When to use it].
  Trigger words: [comma-separated keywords].
---
```

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
