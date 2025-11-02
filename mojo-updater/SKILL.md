---
name: mojo-updater
description: Guide for updating Mojo code to newer versions using test-driven development. Analyzes changelogs, identifies breaking changes, updates dependencies, and systematically fixes code based on deprecations and API changes.
---

# Mojo Version Updater

This skill guides you through updating Mojo code to newer versions using a systematic, test-driven approach. Use when migrating Mojo projects between versions, fixing breaking changes, or adopting new language features.

## Update Workflow

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

1. **Always start with tests passing** - Establish baseline
2. **Update dependencies first** - See what breaks
3. **Fix one pattern at a time** - Easier to track
4. **Run tests frequently** - Fast feedback loop
5. **Document each change** - Helps with future migrations
6. **Keep changelog open** - Reference during fixes
7. **Use version control** - Commit after each successful pattern fix
8. **Test incrementally** - Don't wait until all changes are done

## Resources

- [Mojo Changelog (Unreleased)](https://github.com/modular/modular/blob/main/mojo/docs/changelog.md)
- [Mojo Changelog (Released)](https://github.com/modular/modular/blob/main/mojo/docs/changelog-released.md)
- [Mojo Manual](https://docs.modular.com/mojo/manual/)
- [Example Migration PR](https://github.com/msaelices/mojo-playground/pull/1)

## Example Update Session

```bash
# 1. Fetch changelog
gh api repos/modular/modular/contents/mojo/docs/changelog-released.md --jq '.content' | base64 -d > changelog.txt

# 2. Check current version
mojo --version

# 3. Run tests (baseline)
mojo run test_*.mojo

# 4. Update dependencies
# Edit pixi.toml: mojo = "0.25.6"
pixi install

# 5. Run tests (identify failures)
mojo run test_*.mojo

# 6. Fix each failure based on changelog patterns
# - Remove @value decorators
# - sizeof → size_of
# - Add ImplicitlyCopyable
# - etc.

# 7. Verify all tests pass
mojo run test_*.mojo

# 8. Commit changes
git add -A
git commit -m "Update to Mojo 0.25.6"
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
