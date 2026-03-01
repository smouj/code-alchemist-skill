---
name: code-alchemist
version: 1.2.0
description: Automated code refactoring and architecture transformation engine
author: SMOUJBOT
tags:
  - refactoring
  - code-quality
  - maintainability
  - architecture
  - cleanup
type: transformation
risk: medium
dependencies:
  - eslint
  - prettier
  - typescript
  - @typescript-eslint/parser
  - @typescript-eslint/eslint-plugin
  - biome
  - tsc
  - vitest
  - jest
required_tools:
  - node >= 18
  - git
  - jq
environment_vars:
  - REFACTOR_DRY_RUN: "Set to 'true' to preview changes without applying"
  - REFACTOR_AGGRESSIVE: "Set to 'true' for deeper transformations"
  - REFACTOR_SAFETY_LEVEL: "3 (max) to 1 (min) - controls how conservative changes are"
work_dir: /tmp/code-alchemist-{task_id}
timeout: 180000
---

# Code Alchemist

## Purpose

Transforms messy, legacy, or poorly structured codebases into clean, maintainable architectures through automated refactoring. Targeted at teams needing to:
- Reduce technical debt in production Node.js/TypeScript services
- Standardize code style and patterns across microservices
- Migrate from JavaScript to TypeScript incrementally
- Extract and separate concerns (fat controllers, god objects)
- Convert callback-based code to async/await patterns
- Eliminate code smells: long functions, duplicated logic, deep nesting
- Prepare codebases for team scaling by enforcing conventions

**Real use cases:**
- Pre-sprint cleanup: "Clean up our auth service before new team joins"
- Migration: "Convert all CommonJS requires to ES modules safely"
- Architecture fix: "Split this 2000-line service into domain modules"
- Quality gate: "Ensure all new PRs meet our linting standards"

## Scope

Executes within a target repository to perform controlled, reversible transformations:

### Commands
- `analyze`: Deep codebase analysis, identifies hotspots and refactoring candidates
- `refactor:format`: Apply consistent formatting (Prettier/Biome)
- `refactor:lint`: Fix lintable issues automatically
- `refactor:typescript`: Incremental TypeScript migration, add missing types
- `refactor:extract`: Pull out duplicated code into utilities
- `refactor:layers`: Reorganize into clean architecture layers
- `validate:quality`: Gate check - ensure code meets standards
- `rollback`: Revert last transformation using git stash

### Flags
- `--dry-run`: Preview without modifying files
- `--aggressive`: Enable deeper, riskier transformations
- `--scope=<glob>`: Limit to specific directories (default: entire repo)
- `--exclude=<glob>`: Skip certain patterns (node_modules, tests, migrations)
- `--safety-level=N`: Conservative (1) to aggressive (5)
- `--commit`: Create git commit after successful refactor
- `--skip-tests`: Don't run test suite (use with caution)

## Work Process

### 1. Discovery & Baseline
```bash
# Clone target into isolated workspace
git clone --depth 1 <repo_url> /tmp/code-alchemist-{uuid}

# Capture baseline metrics
./node_modules/.bin/eslint . --format json > baseline-eslint.json
./node_modules/.bin/typescript --noEmit > baseline-tsc.txt
find src -name "*.ts" -o -name "*.js" | wc -l > file-count.txt
```

**Validation:** Ensure repository is healthy (builds, tests pass) before refactoring.

### 2. Analysis Phase
```bash
# Run linter with autofix detection
npx eslint src/ --fix-dry-run --format json > fixable-issues.json

# Identify code smells
grep -r "any" src/ --include="*.ts" > any-usage.txt
grep -r "console.log" src/ --include="*.ts" > debug-statements.txt
find src -name "*.ts" -exec wc -l {} + | sort -rn | head -20 > longest-files.txt
```

**Output:** `analysis-report.json` with:
- Total issues by category
- Top 10 files needing attention
- Estimated refactor time

### 3. Transformation Pipeline
Run sequentially, stopping on failure:

**Step A: Formatting**
```bash
npx biome format --write src/
# OR fallback to prettier
npx prettier --write src/
```

**Step B: Lint Fixes**
```bash
npx eslint src/ --fix
```

**Step C: TypeScript Improvements**
```bash
# Add missing exports/imports
npx tsc --noEmit

# Incremental typing: convert implicit any to unknown first
# Custom script: scripts/improve-types.mjs
node scripts/improve-types.mjs --strategy conservative
```

**Step D: Extract Utilities**
- Detect duplicated code blocks (40+ lines similarity)
- Create extraction plan
- Move to `src/shared/` or `src/utils/`
- Update all imports

**Step E: Layer Separation**
If flag `--layers`:
```
src/
├── domain/     # Business logic, entities
├── application/ # Use cases, services
├── infrastructure/ # DB, external APIs
└── presentation/ # Controllers, routes, UI
```

### 4. Verification
```bash
# Type check
npx tsc --noEmit

# Lint
npx eslint src/

# Tests (unless --skip-tests)
npm test

# Build verification
npm run build
```

All must pass. If any fail, auto-rollback to baseline.

### 5. Commit & Document
```bash
git add .
git commit -m "refactor: automated cleanup by Code Alchemist

- Format: Biome
- Lint: ESLint fixes
- Types: Incremental TypeScript improvements
- Extract: Shared utilities from duplication
- Layers: Reorganized into clean architecture

Stats:
- Files changed: X
- Lines added: Y
- Lines removed: Z
- Estimated debt reduction: [from analysis]"
```

**Log:** `refactor-log.md` with before/after metrics.

## Golden Rules

1. **Never break the build.** If typecheck or tests fail after any step, rollback immediately.
2. **Preserve behavior.** Refactoring changes structure only, no logic changes.
3. **Small commits.** Each transformation creates its own commit (or group), never monolithic changes.
4. **Test first.** Verify tests pass before starting; run tests after each major step.
5. **Respect project configs.** Use existing ESLint/Prettier/TypeScript config; don't impose external rules.
6. **No format wars.** Stick to one formatter (Biome preferred, else Prettier, else ESLint).
7. **Avoid over-engineering.** Don't introduce patterns the codebase doesn't need yet.
8. **Keep tests green.** If `--skip-tests` is not set, failing tests = abort.
9. **Document extraction.** When pulling out utilities, add JSDoc explaining purpose.
10. **Respect boundaries.** Don't cross microservice boundaries; refactor within repo only.

## Examples

### Example 1: Basic cleanup
**User prompt:**
```
code-alchemist refactor:format --scope=src --dry-run
```

**Skill execution:**
```bash
cd /tmp/code-alchemist-task123
npx biome format --write src/
# Dry run: npx biome format src/ --print-diff > format-changes.diff
```

**Output:**
```
Formatting 47 files in src/
- Reformatted: src/auth/AuthService.ts
- Reformatted: src/users/UserController.ts
- Skipped: src/vendor/ (excluded pattern)

Dry-run diff saved to: /tmp/format-changes.diff
```

### Example 2: Full transformation pipeline
**User prompt:**
```
code-alchemist transform --aggressive --commit --safety-level=4
```

**Skill execution:**
```bash
# 1. Analyze
node scripts/analyze.mjs --output analysis.json
# analysis.json: {"hotspots": 3, "duplication": 12%, "any-usage": 45}

# 2. Format
npx prettier --write .

# 3. Lint fix
npx eslint . --fix

# 4. Type improvements
node scripts/improve-types.mjs --strategy aggressive
# Changes: 23 any → unknown, added 15 explicit returns

# 5. Extract duplicated
node scripts/extract-duplicates.mjs --threshold 40
# Created: src/shared/pagination.ts, src/shared/validation.ts

# 6. Verify
npm test && npm run build

# 7. Commit
git add .
git commit -m "refactor: comprehensive cleanup (Code Alchemist automated)"
```

**Output log:**
```
[SUCCESS] Transformation complete
- Files processed: 148
- Lines removed: 234
- Type issues fixed: 67
- Duplicates extracted: 2 modules
- Test status: 42 passed, 0 failed
- Commit: abc1234
```

### Example 3: TypeScript migration
**User prompt:**
```
code-alchemist refactor:typescript --target=js-files --strategy=incremental
```

**Skill execution:**
```bash
# Find .js files without tests first
find src -name "*.js" ! -path "*/test/*" > js-files.txt

# Rename to .ts, fix imports, add basic any types
node scripts/migrate-js-to-ts.mjs --file-list js-files.txt --add-any
# Output: Converted 8 files to .ts
```

**Output:**
```
Migrating:
- src/utils/helpers.js → src/utils/helpers.ts
- src/middleware/auth.js → src/middleware/auth.ts

Next steps:
1. Review added 'any' types (intentional for migration)
2. Run: code-alchemist refactor:typescript --improve-types
3. Gradually replace any with specific types
```

## Rollback Commands

### Manual rollback (last transformation)
```bash
# If commit was made with --commit
git revert HEAD --no-edit
git push origin <branch>

# If no commit but stash exists
git stash list | grep "Code Alchemist"
git stash pop stash@{0}
```

### Skill-provided rollback
```bash
code-alchemist rollback --last
# Equivalent to: git reset --hard ORIG_HEAD
```

### Full repository reset (dangerous)
```bash
code-alchemist rollback --to-baseline --commit-hash=<hash>
# Creates new commit that restores every file to baseline state
```

### Verify rollback safety
```bash
# Check what would be reverted
code-alchemist rollback --preview
# Shows: Files to be restored, lines to be removed/added
```

## Verification Steps

After any refactor operation, run:

```bash
# 1. Type check
npx tsc --noEmit
# Expected: No errors (warnings OK if pre-existing)

# 2. Lint
npx eslint src/ --max-warnings 0
# Expected: 0 warnings

# 3. Tests
npm test -- --coverage
# Expected: 100% pass, coverage >= baseline

# 4. Bundle size
npm run build && du -sh dist/
# Expected: No significant increase (>5%)

# 5. Runtime smoke test
node dist/index.js --smoke-test
# Expected: "OK" within 2 seconds

# 6. Code metrics
node scripts/measure-complexity.mjs
# Expected: Cyclomatic complexity ↓, maintainability index ↑
```

Document results in `refactor-audit.md` with:
- Before/after LOC count
- Issue counts (eslint, tsc)
- Test pass rate
- Any manual review items

## Troubleshooting

### Issue: "Refactor broke tests"
**Fix:** Automatic rollback triggered on test failure. If persisted:
```bash
# Examine changes in failing test file
git diff HEAD~1 src/tests/MyTest.ts
# Manually adjust refactor to preserve test behaviors
code-alchemist refactor:format --exclude="src/tests/**"
```

### Issue: "Cannot fix lint rule XYZ"
**Cause:** Rule requires manual intervention (security, best practice).
**Fix:** 
```bash
npx eslint src/ --rule '{"no-eval": "off"}' --fix
# Or exclude specific file pattern:
code-alchemist refactor:lint --exclude="src/legacy/**"
```

### Issue: "TypeScript errors increased after migration"
**Fix:** Run improvement in phases:
```bash
# Phase 1: Add any types (already done)
# Phase 2: Replace any with unknown
node scripts/improve-types.mjs --strategy replace-any-unknown
# Phase 3: Manual review of any remaining
grep -r ": any" src/
```

### Issue: "Formatting conflicts with Prettier"
**Fix:** Detect project formatter:
```bash
if [ -f .prettierrc ]; then
  FORMATTER="prettier"
elif [ -f biome.json ]; then
  FORMATTER="biome"
else
  FORMATTER="eslint --fix"
fi
```

Set `REFACTOR_FORMATTER=$FORMATTER` environment variable.

### Issue: "Out of memory during analysis"
**Fix:** Process files in batches:
```bash
code-alchemist analyze --batch-size=50 --concurrency=4
```

### Issue: "Git conflicts during rollback"
**Fix:** Use merge strategy:
```bash
git merge --abort
git reset --hard ORIGIN/$(git branch --show-current)
```

## Dependencies

**Install:**
```bash
npm install --save-dev \
  eslint \
  prettier \
  @typescript-eslint/parser \
  @typescript-eslint/eslint-plugin \
  biome \
  typescript \
  vitest \
  jest
```

**Configuration files required:**
- `.eslintrc.js` (or .json/.yaml)
- `.prettierrc` (optional)
- `biome.json` (optional)
- `tsconfig.json`
- `package.json` with scripts: `test`, `build`, `lint`

**No dependencies?** Skill will fail with clear message:
```
ERROR: No ESLint config found. Initialize with: npx eslint --init
```

## Safety Features

- Dry-run default for destructive operations
- Automatic stashing before changes
- Pre-flight checks: repo clean? tests passing?
- Post-flight verification mandatory
- Rollback point created before each phase
- No changes if risk exceeds `--safety-level` threshold

---

```