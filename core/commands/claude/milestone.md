---
description: Validate backlog section completion, then squash+tag or identify gaps
argument-hint: "[backlog-path] [section] [base-branch] [version] [\"validation-prompt\"]"
project-agnostic: true
allowed-tools:
  - Read
  - Edit
  - Bash
  - Grep
  - Glob
  - Task
  - Write
  - AskUserQuestion
---

# Milestone Validation & Release

Validate section completion in backlog, then either release or identify gaps.

**All arguments are optional** with smart defaults.

## Argument Parsing

Parse `$ARGUMENTS` into:

| Variable | Source | Required | Default |
|----------|--------|----------|---------|
| `BACKLOG_PATH` | 1st arg | No | Auto-detect: `BACKLOG.md`, `specs/backlog.md`, or skip |
| `SECTION` | 2nd arg | No | Latest incomplete section, or skip if no backlog |
| `BASE_BRANCH` | 3rd arg | No | `origin/main` (fetches first) |
| `VERSION` | 4th arg (if not quoted) | No | Auto-bump from `VERSION` file or latest git tag |
| `VALIDATION_PROMPT` | Last quoted string `"..."` | No | Auto-derive from checklist |

**Parsing Rules:**
- If an argument starts and ends with `"`, treat as `VALIDATION_PROMPT`
- Version is optional; if 4th arg is quoted, there's no version
- Empty `$ARGUMENTS` triggers full auto-detect mode

## Phase 0: Smart Defaults Resolution

**When NO arguments provided:**

### 0.1 Fetch Remote
```bash
git fetch origin main 2>/dev/null || git fetch origin master 2>/dev/null
```

### 0.2 Determine BASE_BRANCH
```bash
# Try origin/main first, fallback to origin/master
git rev-parse origin/main >/dev/null 2>&1 && echo "origin/main" || echo "origin/master"
```

### 0.3 Auto-Detect BACKLOG_PATH
Search in order:
1. `BACKLOG.md` (root)
2. `specs/backlog.md`
3. `docs/backlog.md`
4. `backlog.md`

If none found → **SKIP backlog validation** (proceed without backlog)

### 0.4 Auto-Detect SECTION (if backlog exists)
Find first incomplete section (has `- [ ]` items). If all complete → use most recent section.

### 0.5 Auto-Detect VERSION
Priority order:
1. `VERSION` file → parse, increment patch: `X.Y.Z` → `X.Y.(Z+1)`, prefix with `v`
2. Latest git tag matching `v*.*.*` → increment patch
3. Default: `v0.1.0`

### 0.6 CHANGELOG Consistency Check (Critical for No-Args Mode)
```bash
# Check if CHANGELOG.md exists and has [Unreleased] content
grep -A 100 "\\[Unreleased\\]" CHANGELOG.md 2>/dev/null | grep -E "^- |^### " | head -20
```

**If CHANGELOG [Unreleased] is EMPTY but commits exist since BASE_BRANCH:**
```
CHANGELOG SYNC REQUIRED

Commits since {BASE_BRANCH}:
  {commit list}

But CHANGELOG.md [Unreleased] section is empty.

Please update CHANGELOG.md with these changes before proceeding.
```
-> **STOP** and wait for user to update changelog.

**If CHANGELOG [Unreleased] has content -> proceed.**

### 0.7 Project-Specific Enforcement

Check if `PROJECT_AGENTS.md` exists in repository root:
```bash
test -f PROJECT_AGENTS.md && echo "exists" || echo "none"
```

**If exists:**
1. Read `PROJECT_AGENTS.md` content
2. Parse and store rules for validation in Phase 4.5
3. Common rules to detect:
   - `no emojis` / `DO NOT use emojis` -> flag emoji restrictions
   - `project-agnostic` / `anonymous` -> flag content anonymity requirements
   - `relative symlinks` -> flag symlink requirements

**Store parsed rules as `PROJECT_RULES` for later validation.**

## Phase 1: Pre-Flight Checks

1. **Git state**: Must be clean (no uncommitted changes)
2. **Commits exist**: Must have commits since `BASE_BRANCH`
3. **CHANGELOG exists**: `CHANGELOG.md` must exist with `[Unreleased]` section
4. **Backlog exists** (if path provided): File at `BACKLOG_PATH` must exist
5. **Section exists** (if backlog provided): Section `SECTION` must be found in backlog

If any fail → STOP with specific error message.

## Phase 2: Extract Checklist (if backlog provided)

1. Read backlog file at `BACKLOG_PATH`
2. Find section matching `SECTION` (patterns: `#### 1.2`, `### Phase 1.2`, `## 1.2`)
3. Extract ALL checklist items until next section:
   - `- [ ]` = unchecked
   - `- [x]` = checked
4. Report: `Found X items (Y checked, Z unchecked)`

**If no backlog:** Skip to Phase 4 with auto-approval path.

## Phase 3: Validate Implementation (if backlog provided)

### If `VALIDATION_PROMPT` provided:
Use it as explicit criteria. Spawn validation agent with prompt:
```
VALIDATE: {VALIDATION_PROMPT}

For each criterion, search codebase and report:
- Status: FOUND | MISSING
- Evidence: file:line references
- Notes: any issues
```

### If NO `VALIDATION_PROMPT`:
For each UNCHECKED item (`- [ ]`):

1. Parse item to identify expected:
   - Files/components (look for nouns)
   - Interfaces/functions (look for code terms)
   - Tests (if mentioned)

2. Search codebase:
   - Grep for key terms
   - Glob for expected file patterns
   - Check test directories

3. Classify:
   - **IMPLEMENTED**: Found despite unchecked
   - **MISSING**: No evidence found

## Phase 4: Decision Gate

### Path A: No Backlog (Changelog-Only Validation)

Display:
```
✅ CHANGELOG Validated

Changes since {BASE_BRANCH}:
- {N} commits
- Key changes: {summary}

CHANGELOG [Unreleased] entries:
{entries}

Release Configuration:
- Base: {BASE_BRANCH}
- Version: {VERSION}
- Commits to squash: N

Proceed with squash + tag? (yes/no)
```

→ On "yes": proceed to squash/tag.

### Path B: Backlog - ALL COMPLETE (all items implemented or checked)

Display:
```
✅ Section {SECTION} COMPLETE

Validated Items:
- [x] Item 1 - evidence: file:line
- [x] Item 2 - evidence: file:line
...

Release Configuration:
- Base: {BASE_BRANCH}
- Version: {VERSION} (or "no tag")
- Commits to squash: N
- CHANGELOG: [Unreleased] → [{VERSION}]

Proceed with squash? (yes/no)
```

**On "yes":**
1. Update backlog: mark all items `[x]`, add checkmark to section header
2. **Update CHANGELOG.md:**
   - Move all entries from `[Unreleased]` to new `[{VERSION}] - {YYYY-MM-DD}` section
   - Keep empty `[Unreleased]` section at top
   - If no VERSION provided, use section name as header
3. Commit: `docs: mark {SECTION} as completed`
4. Create backup: `{branch}-backup/{YYYY}/{MM}/{DD}/001`
5. Soft reset to `BASE_BRANCH`
6. Create squashed commit with comprehensive message
7. If `VERSION`: create annotated tag

→ Proceed to **Phase 5: Push Confirmation**

### Path C: Backlog - INCOMPLETE (missing items)

Display:
```
❌ Section {SECTION} INCOMPLETE

Missing:
1. {item} - {what's missing}
2. {item} - {what's missing}

Implemented but unchecked:
1. {item} - {evidence}

Options:
(A) Create /spec for missing items
(B) Abort - review manually
```

**On "A":** Output spec creation commands:
```
Missing items require specs:

/spec CREATE {item-1-title}
/spec CREATE {item-2-title}
```

**On "B":** STOP cleanly.

## Phase 4.5: PROJECT_AGENTS.md Validation (Pre-Push Gate)

**If `PROJECT_RULES` were parsed in Phase 0.7, validate all changes:**

### 4.5.1 Get Changed Files
```bash
git diff --name-only {BASE_BRANCH}..HEAD
```

### 4.5.2 Validate Against PROJECT_RULES

For each rule detected, run validation:

**Emoji Check (if emoji restriction detected):**
```bash
# Check all changed .md files for emoji characters
git diff {BASE_BRANCH}..HEAD -- '*.md' | grep -P '[\x{1F300}-\x{1F9FF}]' || echo "clean"
```

**Project-Agnostic/Anonymous Check (if anonymity rule detected):**
```bash
# Check for personal identifiers, hardcoded usernames, local paths
git diff {BASE_BRANCH}..HEAD | grep -iE '(/Users/[a-z]+|/home/[a-z]+|@[a-z]+\.com)' || echo "clean"
```

**Symlink Check (if symlink rule detected):**
```bash
# Check for absolute symlinks in changed files
for f in $(git diff --name-only {BASE_BRANCH}..HEAD); do
  if [ -L "$f" ]; then
    target=$(readlink "$f")
    [[ "$target" = /* ]] && echo "ABSOLUTE: $f -> $target"
  fi
done
```

### 4.5.3 Violation Handling

**If ANY violations found:**
```
PROJECT_AGENTS.md VIOLATIONS DETECTED

The following changes violate project-specific rules:

Rule: {rule description}
Violations:
  - {file}: {violation details}
  - {file}: {violation details}

Rule: {rule description}
Violations:
  - {file}: {violation details}

Fix these violations before proceeding with release.
```
-> **STOP** - Do not proceed to Phase 5.

**If NO violations -> proceed to Phase 5.**

## Phase 5: Push Confirmation

After successful squash/tag, display:

```
## Release Prepared Locally

Commit: {sha} {message}
Branch: {branch}
Tag: {VERSION} (if created)
Backup: {backup-branch}

Push to origin?

Commands to execute:
  git push --force-with-lease origin {branch}
  git push origin {VERSION}  # if tag

Proceed with push? (yes/no)
```

**On "yes":**
1. Execute: `git push --force-with-lease origin {branch}`
2. If `VERSION`: Execute: `git push origin {VERSION}`
3. Report success with remote URLs

**On "no":**
Display manual commands and exit:
```
Skipped push. Run manually when ready:
  git push --force-with-lease origin {branch}
  git push origin {VERSION}
```

## Abort Conditions

| Condition | Action |
|-----------|--------|
| Backlog not found (when path specified) | STOP: "File not found: {path}" |
| Section not found (when specified) | STOP: "Section {SECTION} not found. Available: [list]" |
| No commits to squash | STOP: "No commits between {BASE_BRANCH} and HEAD" |
| Git dirty | STOP: "Uncommitted changes. Commit or stash first." |
| CHANGELOG missing | STOP: "CHANGELOG.md not found or missing [Unreleased] section" |
| CHANGELOG empty + commits exist | STOP: "Update CHANGELOG [Unreleased] before proceeding" |
| PROJECT_AGENTS.md violations | STOP: List violations, require fixes before release |
| User declines | STOP cleanly, no changes |
| Push fails | STOP: show error, suggest manual resolution |

## Usage Examples

```bash
# NO ARGUMENTS - full auto mode
# Auto-detects: backlog, section, base branch, version, validates changelog
/milestone

# With backlog and section (base branch auto-detected)
/milestone specs/backlog.md 1.2

# Full release with all args
/milestone specs/backlog.md 1.2 main v0.1.1-alpha

# With validation prompt
/milestone specs/backlog.md 1.2 main v0.1.1-alpha "NoteMeta extended, parser exists, 20 tests pass"

# Without custom prompt (auto-detect from checklist)
/milestone docs/roadmap.md 2.1 main v2.1.0

# Validate only, no version tag
/milestone CHANGELOG.md phase-3 develop

# Quoted prompt without version
/milestone specs/features.md 4.0 main "API endpoints implemented, auth working"
```
