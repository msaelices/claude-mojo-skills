---
name: mojo-updater
description: Guide for updating Mojo code to newer versions using test-driven development. Analyzes changelogs, identifies breaking changes, updates dependencies, and systematically fixes code based on deprecations and API changes.
---

# Mojo Version Updater

This skill guides you through updating Mojo code to newer versions using a systematic, test-driven approach. Use when migrating Mojo projects between versions, fixing breaking changes, or adopting new language features.

## Update Workflow

### Phase 0: Git Branch Setup

**0.1 Check Git Status**

Ensure working directory is clean before starting:

```bash
# Check git status
git status

# Verify no uncommitted changes
git diff --exit-code && git diff --cached --exit-code
```

If there are uncommitted changes, ask user whether to:
- Commit them first
- Stash them
- Abort the update

**0.2 Create Update Branch**

Create a dedicated branch for the version update:

```bash
# Get current branch name
git branch --show-current

# Create and checkout new branch (use target version in name)
git checkout -b update/mojo-0.25.6

# Or for nightly versions
git checkout -b update/mojo-0.25.7-nightly
```

**Branch naming convention:**
- `update/mojo-{version}` - For stable releases (e.g., `update/mojo-0.25.6`)
- `update/mojo-{version}-nightly` - For nightly/dev releases
- `update/mojo-latest` - When updating to latest without specific version

**0.3 Document Starting Point**

```bash
# Record the commit hash before starting
git rev-parse HEAD > .mojo-update-base-commit
```

This allows easy rollback if needed.

### Phase 1: Pre-Update Assessment

**1.1 Identify Current and Target Versions**

Check current version from dependency files:
```bash
# Check pixi.toml
grep -E "(mojo|max)" pixi.toml

# Check current Mojo version
mojo --version
```

Ask user for target version if not specified, or default to latest stable.

Ask for user approval to proceed with update, mentioning the detected current and the target versions and high-level approach.

**1.2 Fetch and Analyze Changelog**

Use `gh` tool to fetch relevant changelog sections:

```bash
# Fetch unreleased changelog (for nightly/dev versions)
gh api repos/modular/modular/contents/mojo/docs/changelog.md --jq '.content' | base64 -d

# Fetch released changelog (for stable versions)
gh api repos/modular/modular/contents/mojo/docs/changelog-released.md --jq '.content' | base64 -d
```

**Extract key information:**
- Breaking changes (deprecated/removed features)
- Renamed functions/methods/types
- Changed import paths
- New required traits or conventions
- Changed function signatures

**1.3 Run Initial Test Suite**

Verify tests pass before making changes:

```bash
# Run tests based on project structure
mojo run test_*.mojo  # Or project-specific test command
# Or if using pytest with Mojo
pytest tests/
```

**Document baseline:** Record which tests pass/fail before migration.

### Phase 2: Dependency Updates

**2.1 Update Dependency Files**

Update `pixi.toml`, `pyproject.toml`, or other dependency files:

```toml
# pixi.toml - Before
[dependencies]
max = ">=25.1.0.dev2025011705"

# pixi.toml - After (targeting v0.25.6+)
[dependencies]
mojo = ">0.25.6.0,<1"
```

**Version patterns:**
- Stable: `mojo = "0.25.6"`
- Nightly: `mojo = ">0.25.6.0.dev2025090505,<1"`
- Range: `mojo = ">=0.25.6,<0.26"`

**2.2 Install Updated Dependencies**

```bash
# Using pixi
pixi install

# Using magic (if applicable)
magic install
```

**2.3 Run Tests to Identify Failures**

```bash
# Run all tests
mojo run test_*.mojo

# Or run specific test
mojo run path/to/test.mojo
```

**Document failures:** Note specific error messages, file paths, and line numbers.

### Phase 3: Systematic Code Migration

Apply fixes based on changelog patterns and test failures. Common migration patterns:

**3.1 Decorator Removals**

```mojo
# BEFORE (v24.x - v25.5)
@value
struct Point:
    var x: Float64
    var y: Float64

# AFTER (v25.6+)
struct Point:
    var x: Float64
    var y: Float64
```

**Pattern:** Remove `@value` decorator (removed in v25.6)

**3.2 Rename: `sizeof` → `size_of`**

```mojo
# BEFORE
from sys import sizeof
var size = sizeof[DType.float32]()

# AFTER
from sys import size_of
var size = size_of[DType.float32]()
```

**Pattern:** Import and calls changed from `sizeof` to `size_of`

**3.3 Rename: `__type_of` → `type_of`**

```mojo
# BEFORE
alias MyType = __type_of(value)

# AFTER
alias MyType = type_of(value)
```

**Pattern:** Remove double underscore prefix

**3.4 Argument Conventions: `owned` → `var` or `deinit`**

```mojo
# BEFORE
fn __del__(owned self):
    pass

fn __moveinit__(out self, owned existing: Self):
    pass

# AFTER
fn __del__(deinit self):
    pass

fn __moveinit__(out self, deinit existing: Self):
    pass
```

**Pattern:** Use `deinit` for destructors and move constructors, `var` for mutable arguments

**3.5 Copyable Traits: Add `ImplicitlyCopyable`**

```mojo
# BEFORE (implicit copying by default)
struct MyStruct:
    var data: Int

# AFTER (v25.6+: must opt-in to implicit copying)
struct MyStruct(ImplicitlyCopyable):
    var data: Int
```

**Pattern:** Add `ImplicitlyCopyable` trait for types that should be implicitly copyable

**3.6 File I/O: String Concatenation in `write()`**

```mojo
# BEFORE
f.write(String("Value: " + String(value) + "\n"))

# AFTER
f.write("Value: ", value, "\n")
```

**Pattern:** `write()` now accepts multiple arguments directly

**3.7 Type Conversions: Explicit `Int`/`UInt` Conversions**

```mojo
# BEFORE (implicit conversion)
var uint_val: UInt = 42
var int_val: Int = uint_val  # Was implicit

# AFTER (explicit conversion required)
var uint_val: UInt = 42
var int_val: Int = Int(uint_val)
```

**Pattern:** Always use explicit `Int(...)` or `UInt(...)` conversions

**3.8 DType Changes: `DType.index` → `DType.int`**

```mojo
# BEFORE
alias IndexType = DType.index

# AFTER
alias IndexType = DType.int
```

**Pattern:** Replace removed `DType.index` with `DType.int`

**3.9 Math Functions: `isqrt` → `rsqrt`**

```mojo
# BEFORE
from math import isqrt
var result = isqrt(value)

# AFTER
from math import rsqrt
var result = rsqrt(value)
```

**Pattern:** Renamed for clarity (reciprocal square root)

**3.10 Trait Access: Qualified References**

```mojo
# BEFORE
trait MyTrait:
    alias Size = 10

    fn method(self):
        var s = Size  # Unqualified

# AFTER
trait MyTrait:
    alias Size = 10

    fn method(self):
        var s = Self.Size  # Qualified
```

**Pattern:** Always use `Self.` prefix in traits

**3.11 Span and StringSlice: `UInt` → `Int` for Length**

```mojo
# BEFORE
var span = Span[Int](ptr, UInt(10))

# AFTER
var span = Span[Int](ptr, 10)  # Int accepted directly
```

**Pattern:** Length parameters now accept `Int` instead of `UInt`

**3.12 Hasher API: Byte Spans**

```mojo
# BEFORE
hasher._update_with_bytes(ptr, length)

# AFTER
hasher._update_with_bytes(Span[Byte](ptr, length))
```

**Pattern:** Takes `Span[Byte]` instead of pointer + length

### Phase 4: Test-Driven Fixes

**4.1 Iterative Fix Process**

For each failing test:

1. **Run the test**
   ```bash
   mojo run path/to/failing_test.mojo
   ```

2. **Analyze the error**
   - Read the error message carefully
   - Identify the line number and context
   - Match against changelog patterns

3. **Apply the fix**
   - Use Edit tool to apply the appropriate migration pattern
   - If unclear, search changelog for the specific error

4. **Re-run the test**
   ```bash
   mojo run path/to/failing_test.mojo
   ```

5. **Verify fix**
   - Test passes → move to next failure
   - Test still fails → analyze new error, repeat

**4.2 Batch Similar Fixes**

If multiple files have the same issue (e.g., `sizeof` → `size_of`):

```bash
# Find all occurrences
grep -r "sizeof" --include="*.mojo" .

# Apply fix to each file
# Use Edit tool for each occurrence
```

**4.3 Run Full Test Suite**

After fixing all failures, run complete test suite:

```bash
# Run all tests
mojo run test_*.mojo

# Or project-specific command
./run_tests.sh
```

### Phase 5: Code Quality and Verification

**5.1 Check for Deprecation Warnings**

Run code and look for deprecation warnings:

```bash
mojo run main.mojo 2>&1 | grep -i "deprecat"
```

Address warnings proactively before they become errors in future versions.

**5.2 Review New Features**

Check changelog for new features that could improve the code:
- New standard library types
- Performance improvements
- Better APIs

**5.3 Update Documentation**

Update comments, READMEs, and inline documentation:
- Note the Mojo version requirement
- Update code examples if syntax changed
- Document any migration-related changes

**5.4 Final Validation**

```bash
# Lint if available
mojo format --check *.mojo

# Run full test suite multiple times
for i in {1..3}; do mojo run test_*.mojo; done

# Build if applicable
mojo build main.mojo
```

### Phase 6: Finalize and Merge

**6.1 Commit All Changes**

Create clear, organized commits for the migration:

```bash
# Review all changes
git status
git diff

# Stage dependency updates
git add pixi.toml pixi.lock
git commit -m "Update Mojo dependency to 0.25.6"

# Stage code fixes (or commit incrementally during Phase 3/4)
git add .
git commit -m "Fix code for Mojo 0.25.6 compatibility

- Remove @value decorators
- Update sizeof → size_of
- Add ImplicitlyCopyable traits
- Update owned → deinit in destructors
- Add explicit Int/UInt conversions
"

# Or if changes were committed incrementally, ensure all are committed
git status  # Should show clean working tree
```

**6.2 Review Commit History**

```bash
# View commits made in this branch
git log origin/main..HEAD --oneline

# Review full diff from main
git diff origin/main
```

**6.3 Push Branch**

```bash
# Push update branch to remote
git push -u origin update/mojo-0.25.6
```

**6.4 Create Pull Request (Optional)**

If working in a team or want code review:

```bash
# Create PR using gh CLI
gh pr create \
  --title "Update to Mojo 0.25.6" \
  --body "## Summary
Updates project to Mojo 0.25.6.

## Changes
- Updated dependencies in pixi.toml
- Removed @value decorators
- Renamed sizeof → size_of
- Added ImplicitlyCopyable traits where needed
- Updated destructors to use deinit convention
- Made Int/UInt conversions explicit

## Testing
- ✅ All tests passing
- ✅ Code builds successfully
- ✅ No deprecation warnings

## Migration Guide
Based on [Mojo 0.25.6 changelog](https://github.com/modular/modular/blob/main/mojo/docs/changelog-released.md)
"
```

Or create PR via web interface using the pushed branch.

**6.5 Merge to Main**

After review (or if working solo):

```bash
# Switch back to main
git checkout main

# Merge update branch (use --no-ff for clear history)
git merge --no-ff update/mojo-0.25.6 -m "Merge Mojo 0.25.6 update"

# Push to main
git push origin main

# Clean up update branch
git branch -d update/mojo-0.25.6
git push origin --delete update/mojo-0.25.6
```

Or merge via PR if created in step 6.4.

**6.6 Clean Up**

```bash
# Remove temporary files
rm .mojo-update-base-commit

# Tag the update if desired
git tag mojo-0.25.6
git push --tags
```

### Rollback Procedure (If Update Fails)

If the update encounters insurmountable issues:

**Option 1: Reset to base commit**
```bash
# Get the starting commit
BASE_COMMIT=$(cat .mojo-update-base-commit)

# Hard reset to before update
git reset --hard $BASE_COMMIT

# Delete update branch
git checkout main
git branch -D update/mojo-0.25.6
```

**Option 2: Revert changes while keeping branch**
```bash
# Revert all commits in the branch
git revert HEAD~n..HEAD  # n = number of commits

# Or reset and re-attempt
git reset --hard origin/main
```

**Option 3: Abandon and start fresh**
```bash
# Switch to main
git checkout main

# Delete update branch
git branch -D update/mojo-0.25.6

# Start over with new approach
git checkout -b update/mojo-0.25.6-v2
```

## Common Migration Patterns by Version

### v24.x → v25.6

**Key changes:**
1. Remove `@value` decorator
2. `sizeof` → `size_of`
3. Add `ImplicitlyCopyable` trait where needed
4. `owned` → `deinit` in destructors
5. Explicit `Int`/`UInt` conversions
6. `DType.index` → `DType.int`
7. Package changes: `max` → `mojo` in dependencies

### v25.6 → v25.7 (nightly)

**Key changes:**
1. `__type_of` → `type_of`
2. `__origin_of` → `origin_of`
3. Tuple syntax changes: `(Int, Float)` is now a tuple instance, not type
4. `ImplicitlyIntable` trait removed
5. `isqrt` → `rsqrt`
6. Origin casting API changes for `LayoutTensor`, `NDBuffer`, `UnsafePointer`
7. `memcpy` deprecations (require keyword arguments)

## Changelog Analysis Strategy

When reading the changelog:

1. **Focus on sections:**
   - ❌ **Removed** - immediate breakage
   - **Language changes** - syntax/semantic changes
   - **Library changes** - API changes
   - **Tooling changes** - build/test changes

2. **Look for patterns:**
   - "renamed from X to Y"
   - "deprecated", "removed"
   - "now requires", "must now"
   - "changed from X to Y"
   - "implicit X is now explicit"

3. **Create a checklist:**
   ```
   [ ] Remove @value decorator
   [ ] Update sizeof → size_of
   [ ] Add ImplicitlyCopyable traits
   [ ] Update pixi.toml dependency
   [ ] Run and fix tests
   ```

## Debugging Migration Issues

**Compiler Error Patterns:**

1. **"cannot find 'X' in scope"**
   → Check if X was renamed or removed in changelog

2. **"missing trait constraint"**
   → New trait requirement (e.g., `ImplicitlyCopyable`)

3. **"cannot implicitly convert"**
   → Add explicit conversion: `Int(value)` or `UInt(value)`

4. **"deprecated: use Y instead"**
   → Direct guidance, replace with Y

5. **"cannot use unqualified reference"**
   → Add `Self.` prefix in trait methods

## Best Practices

1. **Create a dedicated branch** - Always use a separate branch for updates (`update/mojo-{version}`)
2. **Check for clean state** - Ensure no uncommitted changes before starting
3. **Always start with tests passing** - Establish baseline before any changes
4. **Update dependencies first** - See what breaks, commit separately
5. **Fix one pattern at a time** - Easier to track and debug
6. **Run tests frequently** - Fast feedback loop after each fix
7. **Commit incrementally** - Commit after each successful pattern fix or logical group
8. **Document each change** - Clear commit messages help with future migrations
9. **Keep changelog open** - Reference during fixes
10. **Test incrementally** - Don't wait until all changes are done
11. **Create PR for review** - Even solo developers benefit from structured review
12. **Tag releases** - Tag successful migrations for easy reference

## Resources

- [Mojo Changelog (Unreleased)](https://github.com/modular/modular/blob/main/mojo/docs/changelog.md)
- [Mojo Changelog (Released)](https://github.com/modular/modular/blob/main/mojo/docs/changelog-released.md)
- [Mojo Manual](https://docs.modular.com/mojo/manual/)
- [Example Migration PR](https://github.com/msaelices/mojo-playground/pull/1)

## Example Update Session

```bash
# 0. Check git status and create branch
git status
git diff --exit-code && git diff --cached --exit-code
git checkout -b update/mojo-0.25.6
git rev-parse HEAD > .mojo-update-base-commit

# 1. Fetch changelog
gh api repos/modular/modular/contents/mojo/docs/changelog-released.md --jq '.content' | base64 -d > changelog.txt

# 2. Check current version
mojo --version

# 3. Run tests (baseline)
mojo run test_*.mojo

# 4. Update dependencies and commit
# Edit pixi.toml: mojo = "0.25.6"
pixi install
git add pixi.toml pixi.lock
git commit -m "Update Mojo dependency to 0.25.6"

# 5. Run tests (identify failures)
mojo run test_*.mojo

# 6. Fix each failure based on changelog patterns
# - Remove @value decorators
# - sizeof → size_of
# - Add ImplicitlyCopyable
# - etc.

# 7. Verify all tests pass
mojo run test_*.mojo

# 8. Commit code changes
git add .
git commit -m "Fix code for Mojo 0.25.6 compatibility

- Remove @value decorators
- Update sizeof → size_of
- Add ImplicitlyCopyable traits
- Update owned → deinit in destructors
"

# 9. Push branch and create PR
git push -u origin update/mojo-0.25.6
gh pr create --title "Update to Mojo 0.25.6" --body "..."

# 10. After review, merge to main
git checkout main
git merge --no-ff update/mojo-0.25.6
git push origin main

# 11. Clean up
git branch -d update/mojo-0.25.6
git push origin --delete update/mojo-0.25.6
rm .mojo-update-base-commit
```

## Quick Reference: Common Fixes

| Old Pattern | New Pattern | Version |
|------------|-------------|---------|
| `@value struct S` | `struct S` | v25.6+ |
| `sizeof[T]()` | `size_of[T]()` | v25.6+ |
| `owned self` (in `__del__`) | `deinit self` | v25.6+ |
| `DType.index` | `DType.int` | v25.6+ |
| `max = "..."` (pixi.toml) | `mojo = "..."` | v25.6+ |
| Implicit `Int`/`UInt` | `Int(val)`/`UInt(val)` | v25.6+ |
| `__type_of(x)` | `type_of(x)` | v25.7+ |
| `isqrt(x)` | `rsqrt(x)` | v25.7+ |
| `Size` (in trait) | `Self.Size` | v25.6+ |

## Version Notes

This skill is based on Mojo v25.x changelog patterns. For versions beyond v25.7, consult the latest changelog and adapt patterns accordingly.
