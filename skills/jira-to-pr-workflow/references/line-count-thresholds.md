# Line Count Thresholds and PR Size Management

Detailed methodology for tracking PR size, threshold strategy, and PR splitting recommendations.

## Why PR Size Matters

### Research and Best Practices

**Optimal PR Size:**
- **Best Practice:** 200-400 lines of code
- **Maximum Recommended:** 450 lines
- **Exceeds Review Capacity:** 600+ lines

**Impact of PR Size:**
- **<200 lines:** Fast review, high quality feedback, quick merge
- **200-400 lines:** Optimal balance of scope and reviewability
- **400-600 lines:** Reviewers start missing issues, longer review time
- **600+ lines:** Review quality significantly degrades, high risk of bugs

**Study Findings:**
- Reviews of 200 lines take ~30 minutes and catch 95% of issues
- Reviews of 600+ lines take 2+ hours and catch only 60% of issues
- Defect density increases 15% for every 100 lines over 400

### Benefits of Size Limits

**For Reviewers:**
- Focused attention on specific changes
- Easier to understand context and intent
- Less cognitive load, better quality feedback
- Faster turnaround time

**For Authors:**
- Faster feedback cycles
- Clearer scope and purpose
- Easier to address feedback
- Lower risk of merge conflicts

**For Codebase:**
- Incremental changes easier to revert
- Better commit history
- Lower bug introduction rate
- Improved test coverage

## Line Counting Methodology

### Using git diff --stat

**Command:**
```bash
git diff --stat HEAD
```

**Example Output:**
```
src/Views/ProfileView.swift        | 120 ++++++++++++++++++
src/ViewModels/ProfileViewModel.swift | 95 +++++++++++++
src/Services/UserService.swift     | 85 +++++++++++++
tests/ProfileTests.swift            | 80 ++++++++++++
4 files changed, 380 insertions(+)
```

**Parsing Logic:**

```bash
# Extract total insertions
LINES_CHANGED=$(git diff --stat HEAD | tail -1 | \
  grep -oE '[0-9]+ insertions?\(\+\)' | grep -oE '[0-9]+')

# If no insertions, check deletions
if [ -z "$LINES_CHANGED" ]; then
  LINES_CHANGED=$(git diff --stat HEAD | tail -1 | \
    grep -oE '[0-9]+ deletions?\(\-\)' | grep -oE '[0-9]+')
fi

# Default to 0 if still empty
LINES_CHANGED=${LINES_CHANGED:-0}

echo "Total lines changed: $LINES_CHANGED"
```

### What Counts as a Line?

**Included:**
- Code insertions (+)
- Code deletions (-)
- Modified lines (counted as both deletion and insertion)
- Empty lines and whitespace changes
- Comment additions/removals

**Excluded:**
- Auto-generated files (if .gitignore excludes them)
- Binary files (images, fonts, etc.)
- Lock files (package-lock.json, Podfile.lock)

**Rationale:** All changes require cognitive review, even whitespace and comments.

### When to Count

**Trigger Points:**

1. **After Each Task Completion:**
   - Monitor TodoWrite updates
   - Count when task status changes to "completed"
   - Store cumulative count

2. **Before Commit:**
   - Final count before creating commit
   - Verify threshold compliance

3. **During Implementation (Optional):**
   - Every 100 lines (for incremental reviews)
   - Provides early warning

**Continuous Tracking:**

```python
def track_line_count_continuously():
    """
    Monitor line count during feature-marker implementation.

    Hooks into TodoWrite completions.
    """

    # Load previous count
    previous_count = load_line_count()

    # Get current count
    current_count = count_lines_changed()

    # Calculate delta
    delta = current_count - previous_count

    print(f"Lines changed this task: {delta}")
    print(f"Total lines changed: {current_count}")

    # Check thresholds
    check_thresholds(current_count)

    # Save for next iteration
    save_line_count(current_count)
```

## Threshold Strategy

### Dual Threshold System

**350 Lines - Soft Warning:**
- **Purpose:** Early warning, prepare for potential split
- **Action:** Display warning, continue automatically
- **User Impact:** Informational only, no blocking

**450 Lines - Hard Stop:**
- **Purpose:** Enforce quality limit, require explicit approval
- **Action:** Block further implementation, require user decision
- **User Impact:** Must approve to continue or agree to split

### Threshold Implementation

```python
def check_thresholds(lines_changed):
    """
    Check line count against thresholds and take action.
    """

    SOFT_THRESHOLD = 350
    HARD_THRESHOLD = 450

    if lines_changed < SOFT_THRESHOLD:
        # Under soft threshold, no action needed
        return True

    elif SOFT_THRESHOLD <= lines_changed < HARD_THRESHOLD:
        # Soft warning
        display_soft_warning(lines_changed)
        return True  # Continue automatically

    else:  # lines_changed >= HARD_THRESHOLD
        # Hard stop
        approved = display_hard_stop(lines_changed)
        return approved  # Only continue if user approves
```

### Soft Warning Display

```python
def display_soft_warning(lines_changed):
    """
    Display soft warning at 350 lines.
    """

    print(f"""
⚠️  Approaching PR size limit: {lines_changed} lines (target: ≤450 lines)

File breakdown:
""")

    # Show file-by-file breakdown
    display_file_breakdown()

    print("""
Continuing automatically. Consider splitting if exceeds 450 lines.
""")
```

**Example Output:**
```
⚠️  Approaching PR size limit: 380 lines (target: ≤450 lines)

File breakdown:
  src/Views/ProfileView.swift        | 120 +++++
  src/ViewModels/ProfileViewModel.swift | 95 +++++
  src/Services/UserService.swift     | 85 +++++
  tests/ProfileTests.swift            | 80 +++++

Continuing automatically. Consider splitting if exceeds 450 lines.
```

### Hard Stop Display

```python
def display_hard_stop(lines_changed):
    """
    Display hard stop at 450 lines and require approval.
    """

    print(f"""
🛑 PR size limit reached: {lines_changed} lines (target: ≤450 lines)

File breakdown:
""")

    # Show file-by-file breakdown
    display_file_breakdown()

    print(f"""
This PR exceeds the recommended size for review quality.

Research shows:
- PRs over 450 lines take 2+ hours to review
- Reviewers catch only 60% of issues (vs 95% for <400 lines)
- Defect density increases 15% per 100 lines over 400

Recommendations:
1. Split into multiple PRs (recommended)
2. Continue with current size (not recommended)

Options:
  [split] Split into multiple PRs (recommended)
  [continue] Continue anyway
  [abort] Abort and manually reorganize
""")

    while True:
        choice = input("Your choice [split/continue/abort]: ").strip().lower()

        if choice == 'split':
            return handle_split_request(lines_changed)
        elif choice == 'continue':
            print("⚠️  Continuing with oversized PR (review quality may suffer)")
            log_size_exception(lines_changed)
            return True
        elif choice == 'abort':
            print("Workflow aborted. State preserved for manual reorganization.")
            return False
        else:
            print("Invalid choice. Please enter 'split', 'continue', or 'abort'")
```

## File-Level Breakdown

### Display File Breakdown

```python
def display_file_breakdown():
    """
    Display per-file line changes.
    """

    # Get file stats from git
    output = subprocess.check_output(
        ['git', 'diff', '--stat', 'HEAD'],
        text=True
    )

    # Parse and display
    lines = output.strip().split('\n')

    # Skip summary line (last line)
    file_lines = lines[:-1]

    for line in file_lines:
        print(f"  {line}")
```

### Identify Large Files

```python
def identify_large_files(threshold=100):
    """
    Identify files with more than threshold lines changed.

    Useful for suggesting split points.
    """

    output = subprocess.check_output(
        ['git', 'diff', '--numstat', 'HEAD'],
        text=True
    )

    large_files = []

    for line in output.strip().split('\n'):
        if not line:
            continue

        parts = line.split('\t')
        if len(parts) != 3:
            continue

        added = int(parts[0]) if parts[0] != '-' else 0
        deleted = int(parts[1]) if parts[1] != '-' else 0
        filename = parts[2]

        total_changes = added + deleted

        if total_changes >= threshold:
            large_files.append({
                'file': filename,
                'added': added,
                'deleted': deleted,
                'total': total_changes
            })

    # Sort by total changes (largest first)
    large_files.sort(key=lambda x: x['total'], reverse=True)

    return large_files
```

## PR Splitting Strategies

### When to Split

**Criteria for Splitting:**
- Total lines ≥ 450
- Multiple logical features/changes in one PR
- Changes affect multiple unrelated areas
- Mix of refactoring and new features

**When NOT to Split:**
- Changes are tightly coupled
- Splitting would break functionality
- Intermediate states are broken/untestable
- High merge conflict risk

### Splitting Patterns

**Pattern 1: Feature-Based Split**

Split by logical features or user stories.

**Example:**
- PR 1: User profile view (120 lines)
- PR 2: Profile editing (150 lines)
- PR 3: Profile photo upload (180 lines)

**Benefits:**
- Each PR delivers value independently
- Clear scope and purpose
- Easy to review and test

**Pattern 2: Layer-Based Split**

Split by architectural layers.

**Example:**
- PR 1: Data models and migrations (150 lines)
- PR 2: API endpoints and services (200 lines)
- PR 3: UI components (180 lines)

**Benefits:**
- Changes reviewed by appropriate experts
- Bottom-up integration (models → API → UI)
- Lower risk of merge conflicts

**Pattern 3: Refactor-Then-Feature**

Split refactoring from new functionality.

**Example:**
- PR 1: Refactor existing auth system (200 lines)
- PR 2: Add biometric authentication (250 lines)

**Benefits:**
- Refactoring reviewed independently
- New feature built on clean foundation
- Easier to revert if needed

**Pattern 4: Component-Based Split**

Split by independent components.

**Example:**
- PR 1: User profile component (180 lines)
- PR 2: Settings component (200 lines)
- PR 3: Integration and navigation (120 lines)

**Benefits:**
- Components developed in parallel
- Each PR fully tests its component
- Clear boundaries

### Automated Split Suggestions

```python
def suggest_split_strategy(large_files, total_lines):
    """
    Analyze changes and suggest optimal split strategy.
    """

    # Analyze file patterns
    file_categories = categorize_files(large_files)

    print(f"""
PR Split Suggestions ({total_lines} total lines)

File Categories:
""")

    for category, files in file_categories.items():
        total_category_lines = sum(f['total'] for f in files)
        print(f"  {category}: {len(files)} files, {total_category_lines} lines")

    print("\nRecommended Split Strategy:")

    # Pattern 1: Feature-based (if multiple features detected)
    features = detect_features_from_tasks()
    if len(features) > 1:
        print("\n✓ Feature-Based Split (Recommended)")
        for i, feature in enumerate(features, 1):
            print(f"  PR {i}: {feature['name']} ({feature['lines']} lines)")
            print(f"    Files: {', '.join(feature['files'])}")

    # Pattern 2: Layer-based (if changes span multiple layers)
    elif 'models' in file_categories and 'views' in file_categories and 'services' in file_categories:
        print("\n✓ Layer-Based Split (Recommended)")
        print(f"  PR 1: Data Models ({file_categories['models']['lines']} lines)")
        print(f"  PR 2: Services/API ({file_categories['services']['lines']} lines)")
        print(f"  PR 3: UI/Views ({file_categories['views']['lines']} lines)")

    # Pattern 3: Component-based (if independent components detected)
    elif detect_independent_components(large_files):
        components = group_by_component(large_files)
        print("\n✓ Component-Based Split (Recommended)")
        for i, (component, files) in enumerate(components.items(), 1):
            total = sum(f['total'] for f in files)
            print(f"  PR {i}: {component} ({total} lines)")

    # Pattern 4: File-size-based (fallback)
    else:
        print("\n✓ File-Size-Based Split (Fallback)")
        print("  Split largest files into separate PRs:")
        cumulative = 0
        pr_num = 1
        pr_files = []

        for f in large_files:
            if cumulative + f['total'] > 400:
                print(f"  PR {pr_num}: {', '.join([file['file'] for file in pr_files])} ({cumulative} lines)")
                pr_num += 1
                cumulative = 0
                pr_files = []

            pr_files.append(f)
            cumulative += f['total']

        if pr_files:
            print(f"  PR {pr_num}: {', '.join([file['file'] for file in pr_files])} ({cumulative} lines)")
```

### File Categorization

```python
def categorize_files(files):
    """
    Categorize files by type/layer.
    """

    categories = {
        'models': [],
        'views': [],
        'viewmodels': [],
        'services': [],
        'tests': [],
        'other': []
    }

    for f in files:
        filename = f['file'].lower()

        if 'model' in filename or '/models/' in filename:
            categories['models'].append(f)
        elif 'view' in filename or '/views/' in filename:
            categories['views'].append(f)
        elif 'viewmodel' in filename or '/viewmodels/' in filename:
            categories['viewmodels'].append(f)
        elif 'service' in filename or '/services/' in filename:
            categories['services'].append(f)
        elif 'test' in filename or '/tests/' in filename:
            categories['tests'].append(f)
        else:
            categories['other'].append(f)

    # Remove empty categories
    return {k: v for k, v in categories.items() if v}
```

## Configuration Overrides

### Custom Thresholds

**Configuration File:** `.jira-to-pr-workflow.json`

```json
{
  "line_limit_soft": 300,
  "line_limit_hard": 400,
  "auto_review_interval": 80
}
```

**Load Configuration:**

```python
def load_line_count_config():
    """
    Load line count configuration from project file.
    """

    config_file = '.jira-to-pr-workflow.json'

    if not os.path.exists(config_file):
        # Use defaults
        return {
            'soft_limit': 350,
            'hard_limit': 450,
            'review_interval': 100
        }

    with open(config_file, 'r') as f:
        config = json.load(f)

    return {
        'soft_limit': config.get('line_limit_soft', 350),
        'hard_limit': config.get('line_limit_hard', 450),
        'review_interval': config.get('auto_review_interval', 100)
    }
```

### Per-Project Adjustments

**Rationale for Custom Limits:**

- **Stricter Limits (300/400):**
  - Critical production systems
  - High-risk areas (security, payments)
  - Junior team members
  - Complex domain logic

- **Relaxed Limits (400/550):**
  - Prototypes and experiments
  - Senior team members
  - Well-tested areas
  - Automated code generation

**Example Configuration:**

```json
{
  "line_limit_soft": 300,
  "line_limit_hard": 400,
  "reason": "Production payment processing system - stricter limits for safety",
  "auto_review_interval": 80
}
```

## Exclusions and Special Cases

### Auto-Generated Files

**Exclude from Count:**
- Xcode project files (`.pbxproj`)
- Package lock files (`Package.resolved`, `Podfile.lock`)
- Generated Swift code (`*.generated.swift`)
- Localization files (`.strings`, `.stringsdict`)

**Implementation:**

```bash
# Count only non-generated files
git diff --stat HEAD -- \
  ':(exclude)*.pbxproj' \
  ':(exclude)*.lock' \
  ':(exclude)*.generated.swift' \
  ':(exclude)*.strings'
```

### Large Test Files

**Special Handling:**

Test files often exceed normal limits due to multiple test cases.

**Strategy:**
- Count test files separately
- Apply 1.5x multiplier to limits for test-only PRs
- Example: 450 line limit → 675 for tests

**Detection:**

```python
def adjust_limits_for_tests(changed_files, soft_limit, hard_limit):
    """
    Adjust limits if PR is primarily tests.
    """

    test_files = [f for f in changed_files if 'test' in f.lower()]
    test_lines = sum_lines_for_files(test_files)
    total_lines = sum_lines_for_files(changed_files)

    # If >80% of changes are tests, apply multiplier
    if test_lines / total_lines > 0.8:
        print("ℹ️  Test-heavy PR detected, applying 1.5x limit multiplier")
        return soft_limit * 1.5, hard_limit * 1.5

    return soft_limit, hard_limit
```

## State Persistence

### Save Line Count State

```python
def save_line_count(count, feature_slug):
    """
    Save current line count for checkpoint resume.
    """

    state_file = f".claude/feature-state/{feature_slug}/line-count.json"

    state = {
        'current_count': count,
        'soft_threshold': 350,
        'hard_threshold': 450,
        'warnings_issued': count >= 350,
        'limit_exceeded': count >= 450,
        'last_updated': datetime.utcnow().isoformat() + 'Z'
    }

    with open(state_file, 'w') as f:
        json.dump(state, f, indent=2)
```

### Load on Resume

```python
def load_line_count(feature_slug):
    """
    Load line count state from checkpoint.
    """

    state_file = f".claude/feature-state/{feature_slug}/line-count.json"

    if not os.path.exists(state_file):
        return 0

    with open(state_file, 'r') as f:
        state = json.load(f)

    return state['current_count']
```

## Best Practices Summary

1. **Count Early and Often:** Track line count after each task completion

2. **Use Dual Thresholds:** Soft warning (350) + hard stop (450)

3. **Show File Breakdown:** Help users understand what's contributing to size

4. **Suggest Splits:** Provide actionable recommendations for oversized PRs

5. **Respect User Choice:** Allow override with approval for edge cases

6. **Exclude Generated Files:** Don't count auto-generated code

7. **Adjust for Tests:** Apply multiplier for test-heavy PRs

8. **Persist State:** Save line count for checkpoint resume

9. **Log Exceptions:** Record when users override limits

10. **Educate:** Explain why limits matter (review quality, defect rates)
