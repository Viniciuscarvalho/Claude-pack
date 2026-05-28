# Code Review Orchestration

Detailed patterns for detecting, invoking, and aggregating feedback from code review agents.

## Agent Detection Logic

### Required Agents

From requirements specification:
- `code-reviewer` - General code review (from pr-review-toolkit)
- `swift-expert` - Swift-specific deep review
- `swiftui-specialist` - SwiftUI performance and architecture
- `swift-reviewer` - Swift best practices and idioms

### Detection Strategy

**Graceful Degradation Principle:** Agent unavailability should NEVER block the workflow.

**Detection Pattern:**

```python
def detect_available_agents():
    """
    Detect which code review agents are available.

    Returns:
      dict: {agent_name: bool} availability map
    """

    required_agents = [
        'code-reviewer',
        'swift-expert',
        'swiftui-specialist',
        'swift-reviewer'
    ]

    available_agents = {}

    for agent in required_agents:
        try:
            # Attempt to invoke agent with --check flag
            result = invoke_skill(agent, args="--check")
            available_agents[agent] = True
            print(f"✓ {agent} available")
        except AgentNotFoundError:
            available_agents[agent] = False
            print(f"⚠️  {agent} not available, skipping")
        except Exception as e:
            # Other errors also mark as unavailable
            available_agents[agent] = False
            print(f"⚠️  {agent} error: {e}, skipping")

    # Report summary
    available_count = sum(available_agents.values())
    total_count = len(required_agents)

    if available_count == 0:
        print(f"""
⚠️  Warning: No code review agents available

Code review will be skipped. Consider installing:
  - code-reviewer (general review)
  - swift-expert (Swift deep review)
  - swiftui-specialist (SwiftUI review)
  - swift-reviewer (Swift best practices)

Continuing without code review...
""")
    elif available_count < total_count:
        unavailable = [name for name, avail in available_agents.items() if not avail]
        print(f"""
ℹ️  Info: {available_count}/{total_count} agents available

Missing: {', '.join(unavailable)}

Continuing with available agents...
""")
    else:
        print(f"✓ All {total_count} code review agents available")

    return available_agents
```

### Agent Invocation Testing

**Before full review, test agent availability:**

```bash
# For each agent, try a minimal invocation
Skill(skill="code-reviewer", args="--check")  # Test if exists

# If successful, agent is available
# If error, agent is unavailable
```

## Parallel Agent Invocation Patterns

### Incremental Review (Phase 3)

**Purpose:** Lightweight, non-blocking review during implementation

**When:** Every ~100 lines OR 2-3 tasks completed

**Strategy:**

```python
def run_incremental_review(changed_files, available_agents):
    """
    Run lightweight review in parallel during implementation.

    Non-blocking: Launch agents without waiting for results.
    """

    if not available_agents:
        return  # Skip if no agents available

    print(f"🔍 Running incremental code review on {len(changed_files)} files...")

    review_results = []

    # Launch agents in parallel
    for agent_name, is_available in available_agents.items():
        if not is_available:
            continue

        # Invoke asynchronously
        task = invoke_agent_async(
            agent=agent_name,
            mode="quick",
            files=changed_files,
            focus="recent changes only"
        )

        review_results.append({
            'agent': agent_name,
            'task': task,
            'status': 'running'
        })

    # Don't wait for results - continue implementation
    # Results will be collected for final aggregation

    return review_results
```

**Async Invocation Pattern:**

```python
def invoke_agent_async(agent, mode, files, focus):
    """
    Invoke agent asynchronously (non-blocking).

    Uses Task tool with run_in_background=True.
    """

    prompt = f"""
Review the following files with {mode} mode:

Files: {', '.join(files)}
Focus: {focus}

Provide feedback with:
- Issue description
- File and line number
- Severity (Critical/Important/Minor)
- Confidence score (0-100)
"""

    task = Task(
        subagent_type=agent,
        description=f"{mode} review by {agent}",
        prompt=prompt,
        run_in_background=True  # Non-blocking
    )

    return task
```

### Final Review (Phase 4)

**Purpose:** Comprehensive, blocking review of entire changeset

**When:** After tests pass, before commit

**Strategy:**

```python
def run_final_review(all_changed_files, available_agents):
    """
    Run comprehensive review in parallel (blocking).

    Wait for all agents to complete before proceeding.
    """

    if not any(available_agents.values()):
        print("⚠️  Skipping code review (no agents available)")
        return []

    print(f"🔍 Running final code review with {sum(available_agents.values())} agents...")

    review_tasks = []

    # Launch all available agents in parallel
    for agent_name, is_available in available_agents.items():
        if not is_available:
            continue

        print(f"  Launching {agent_name}...")

        task = invoke_agent_sync(
            agent=agent_name,
            mode="full",
            files=all_changed_files,
            focus="comprehensive review"
        )

        review_tasks.append({
            'agent': agent_name,
            'task': task
        })

    # Wait for all tasks to complete
    print(f"  Waiting for {len(review_tasks)} agents to complete...")

    results = []
    for item in review_tasks:
        agent_name = item['agent']
        task = item['task']

        print(f"  Waiting for {agent_name}...")
        result = task.wait()  # Blocking

        results.append({
            'agent': agent_name,
            'feedback': result,
            'status': 'completed'
        })

        print(f"  ✓ {agent_name} complete")

    return results
```

**Sync Invocation Pattern:**

```python
def invoke_agent_sync(agent, mode, files, focus):
    """
    Invoke agent synchronously (blocking).

    Uses Task tool with run_in_background=False (default).
    """

    prompt = f"""
Review the following files with {mode} review:

Files: {', '.join(files)}
Focus: {focus}

Provide comprehensive feedback covering:
1. **Architecture & Design**
   - Code organization
   - Design patterns
   - SOLID principles
   - Separation of concerns

2. **Swift/SwiftUI Specifics**
   - Modern Swift idioms
   - SwiftUI best practices
   - Performance considerations
   - Memory management

3. **Security & Safety**
   - Input validation
   - Error handling
   - Force unwraps and optionals
   - Data privacy

4. **Code Quality**
   - Readability
   - Maintainability
   - Test coverage
   - Documentation

For each issue found, provide:
- Clear description
- File path and line number
- Severity: Critical (90-100) / Important (80-89) / Minor (51-79)
- Confidence score: 0-100
- Suggested fix
"""

    task = Task(
        subagent_type=agent,
        description=f"{mode} review by {agent}",
        prompt=prompt,
        run_in_background=False  # Blocking
    )

    return task
```

## Feedback Aggregation

### Collect All Feedback

```python
def collect_all_feedback(incremental_results, final_results):
    """
    Collect feedback from both incremental and final reviews.
    """

    all_feedback = []

    # Collect incremental review results
    for item in incremental_results:
        if item['status'] == 'completed':
            all_feedback.append({
                'agent': item['agent'],
                'mode': 'incremental',
                'feedback': item['task'].get_result(),
                'timestamp': item['timestamp']
            })

    # Collect final review results
    for item in final_results:
        all_feedback.append({
            'agent': item['agent'],
            'mode': 'final',
            'feedback': item['feedback'],
            'timestamp': datetime.utcnow()
        })

    return all_feedback
```

### Parse Agent Output

**Expected agent output format:**

```markdown
## Review Results

### Critical Issues (90-100 confidence)
- **File:** `ProfileView.swift:45`
  **Issue:** Force unwrap of optional without nil check
  **Severity:** Critical
  **Confidence:** 95
  **Fix:** Use optional binding or guard statement

### Important Issues (80-89 confidence)
- **File:** `ProfileViewModel.swift:15`
  **Issue:** Missing @MainActor annotation
  **Severity:** Important
  **Confidence:** 85
  **Fix:** Add @MainActor to class declaration

### Minor Suggestions (51-79 confidence)
- **File:** `UserService.swift:30`
  **Issue:** Consider using guard instead of if-let
  **Severity:** Minor
  **Confidence:** 60
  **Fix:** Replace if-let with guard for early return
```

**Parsing Logic:**

```python
def parse_agent_feedback(feedback_text):
    """
    Parse agent feedback text into structured data.

    Returns: List of issues with metadata
    """

    issues = []

    # Regex patterns
    file_pattern = r'\*\*File:\*\* `([^:]+):(\d+)`'
    issue_pattern = r'\*\*Issue:\*\* (.+?)(?=\n\*\*|$)'
    severity_pattern = r'\*\*Severity:\*\* (Critical|Important|Minor)'
    confidence_pattern = r'\*\*Confidence:\*\* (\d+)'
    fix_pattern = r'\*\*Fix:\*\* (.+?)(?=\n\n|\n\*\*|$)'

    # Split by issue blocks
    blocks = re.split(r'\n- \*\*File:', feedback_text)

    for block in blocks[1:]:  # Skip first empty split
        # Extract fields
        file_match = re.search(file_pattern, block)
        issue_match = re.search(issue_pattern, block)
        severity_match = re.search(severity_pattern, block)
        confidence_match = re.search(confidence_pattern, block)
        fix_match = re.search(fix_pattern, block, re.DOTALL)

        if file_match and issue_match:
            issue = {
                'file': file_match.group(1),
                'line': int(file_match.group(2)),
                'description': issue_match.group(1).strip(),
                'severity': severity_match.group(1) if severity_match else 'Minor',
                'confidence': int(confidence_match.group(1)) if confidence_match else 50,
                'fix': fix_match.group(1).strip() if fix_match else None
            }
            issues.append(issue)

    return issues
```

### Deduplicate Issues

```python
def deduplicate_issues(all_issues):
    """
    Remove duplicate issues reported by multiple agents.

    Deduplication logic:
    - Same file and line → Duplicate
    - Same description (fuzzy match) → Duplicate
    - Keep issue with highest confidence
    """

    unique_issues = []
    seen = set()

    # Sort by confidence (highest first)
    sorted_issues = sorted(all_issues, key=lambda x: x['confidence'], reverse=True)

    for issue in sorted_issues:
        # Create unique key
        key = (issue['file'], issue['line'])

        if key not in seen:
            unique_issues.append(issue)
            seen.add(key)
        else:
            # Duplicate found, merge if needed
            # Find existing issue
            existing = next(i for i in unique_issues if (i['file'], i['line']) == key)

            # If new issue has different description, append
            if not fuzzy_match(issue['description'], existing['description']):
                # Different issue at same location, keep both
                unique_issues.append(issue)

    return unique_issues


def fuzzy_match(str1, str2, threshold=0.8):
    """
    Fuzzy match two strings using Levenshtein distance.

    Returns: True if similarity >= threshold
    """

    # Simple implementation (can use difflib.SequenceMatcher)
    from difflib import SequenceMatcher
    ratio = SequenceMatcher(None, str1, str2).ratio()
    return ratio >= threshold
```

### Group by Severity

```python
def group_by_severity(issues):
    """
    Group issues by severity based on confidence score.

    Severity Mapping:
    - Critical: confidence >= 90
    - Important: 80 <= confidence < 90
    - Minor: 51 <= confidence < 80
    """

    grouped = {
        'critical': [],
        'important': [],
        'minor': []
    }

    for issue in issues:
        confidence = issue['confidence']

        if confidence >= 90:
            grouped['critical'].append(issue)
        elif confidence >= 80:
            grouped['important'].append(issue)
        elif confidence >= 51:
            grouped['minor'].append(issue)
        # Ignore issues with confidence < 51

    return grouped
```

## Report Formatting

### Console Report Template

```python
def format_review_report(grouped_issues, agents_executed):
    """
    Format review results for console output.
    """

    critical_count = len(grouped_issues['critical'])
    important_count = len(grouped_issues['important'])
    minor_count = len(grouped_issues['minor'])
    total_count = critical_count + important_count + minor_count

    report = f"""
## Code Review Results

### Summary
- Total Issues: {total_count}
- Critical: {critical_count}
- Important: {important_count}
- Minor: {minor_count}

"""

    # Critical Issues
    if critical_count > 0:
        report += "### Critical Issues (Must Fix)\n\n"
        for issue in grouped_issues['critical']:
            report += f"- **[{issue['agent']}]** {issue['description']}\n"
            report += f"  File: `{issue['file']}:{issue['line']}`\n"
            if issue['fix']:
                report += f"  Fix: {issue['fix']}\n"
            report += "\n"

    # Important Issues
    if important_count > 0:
        report += "### Important Issues (Recommended)\n\n"
        for issue in grouped_issues['important']:
            report += f"- **[{issue['agent']}]** {issue['description']}\n"
            report += f"  File: `{issue['file']}:{issue['line']}`\n"
            if issue['fix']:
                report += f"  Fix: {issue['fix']}\n"
            report += "\n"

    # Minor Suggestions
    if minor_count > 0:
        report += "### Minor Suggestions\n\n"
        for issue in grouped_issues['minor']:
            report += f"- **[{issue['agent']}]** {issue['description']}\n"
            report += f"  File: `{issue['file']}:{issue['line']}`\n"
            if issue['fix']:
                report += f"  Fix: {issue['fix']}\n"
            report += "\n"

    # Footer
    report += "---\n"
    report += f"Agents Executed: {', '.join(agents_executed)}\n"
    report += f"Review Coverage: {calculate_coverage()}%\n"

    return report
```

### Markdown Report for PR

```python
def format_pr_review_summary(grouped_issues, agents_executed):
    """
    Format review summary for PR description.
    """

    critical_count = len(grouped_issues['critical'])
    important_count = len(grouped_issues['important'])
    minor_count = len(grouped_issues['minor'])

    # Overall status
    if critical_count == 0:
        status = "✅ **Passed:**"
    else:
        status = "⚠️  **Critical issues found:**"

    summary = f"""
## Code Review Summary

{status} {', '.join(agents_executed)}

**Issues Found:**
- Critical: {critical_count} (all resolved)
- Important: {important_count} ({count_resolved(grouped_issues['important'])} resolved)
- Minor: {minor_count} ({count_addressed(grouped_issues['minor'])} addressed)

"""

    # Add details for critical/important if any
    if critical_count > 0:
        summary += "\n**Critical Issues (All Resolved):**\n"
        for issue in grouped_issues['critical']:
            summary += f"- [{issue['agent']}] {issue['description']} (`{issue['file']}:{issue['line']}`)\n"

    if important_count > 0:
        summary += "\n**Important Issues:**\n"
        for issue in grouped_issues['important'][:3]:  # Show top 3
            summary += f"- [{issue['agent']}] {issue['description']} (`{issue['file']}:{issue['line']}`)\n"

        if important_count > 3:
            summary += f"\n_... and {important_count - 3} more (see review-feedback.md)_\n"

    return summary
```

### Save to File

```python
def save_review_feedback(grouped_issues, feature_slug):
    """
    Save detailed review feedback to file for user reference.
    """

    output_file = f".claude/feature-state/{feature_slug}/review-feedback.md"

    with open(output_file, 'w') as f:
        f.write("# Code Review Feedback\n\n")
        f.write(f"Generated: {datetime.utcnow().isoformat()}Z\n\n")

        # Write all issues with full details
        for severity in ['critical', 'important', 'minor']:
            issues = grouped_issues[severity]
            if not issues:
                continue

            f.write(f"## {severity.capitalize()} Issues\n\n")

            for issue in issues:
                f.write(f"### {issue['file']}:{issue['line']}\n\n")
                f.write(f"**Agent:** {issue['agent']}\n")
                f.write(f"**Severity:** {issue['severity']}\n")
                f.write(f"**Confidence:** {issue['confidence']}\n\n")
                f.write(f"**Issue:**\n{issue['description']}\n\n")

                if issue['fix']:
                    f.write(f"**Suggested Fix:**\n{issue['fix']}\n\n")

                f.write("---\n\n")

    print(f"✓ Review feedback saved to {output_file}")
```

## User Decision Handling

### Present Review and Get Decision

```python
def get_user_decision_on_review(report, critical_count):
    """
    Present review report and get user decision.

    Returns: 'y' (continue), 'n' (abort), or 'fix' (pause for fixes)
    """

    print(report)

    if critical_count > 0:
        print(f"""
⚠️  {critical_count} critical issue(s) found

It is strongly recommended to fix critical issues before committing.
""")

    print("""
Continue to commit?
  [y] Yes - Continue despite issues (will be logged)
  [n] No - Abort workflow (preserve state)
  [fix] Fix - Pause for manual fixes, resume after
""")

    while True:
        decision = input("Your choice [y/n/fix]: ").strip().lower()

        if decision in ['y', 'n', 'fix']:
            return decision
        else:
            print("Invalid choice. Please enter 'y', 'n', or 'fix'")
```

### Handle Decision

```python
def handle_review_decision(decision, grouped_issues, feature_slug):
    """
    Handle user decision on review results.
    """

    if decision == 'y':
        # Continue despite issues
        print("✓ Continuing to commit (review logged)")

        # Log decision
        log_file = f".claude/feature-state/{feature_slug}/review-decision.md"
        with open(log_file, 'w') as f:
            f.write(f"# Review Decision\n\n")
            f.write(f"Decision: Continue despite issues\n")
            f.write(f"Timestamp: {datetime.utcnow().isoformat()}Z\n\n")
            f.write(f"Issues accepted:\n")
            for severity, issues in grouped_issues.items():
                f.write(f"- {severity.capitalize()}: {len(issues)}\n")

        return True  # Continue to next phase

    elif decision == 'n':
        # Abort workflow
        print("✗ Workflow aborted (state preserved)")
        print(f"Review feedback saved to: .claude/feature-state/{feature_slug}/review-feedback.md")
        return False  # Exit skill

    elif decision == 'fix':
        # Pause for fixes
        print(f"""
Workflow paused for manual fixes.

Review feedback saved to:
  .claude/feature-state/{feature_slug}/review-feedback.md

To resume after fixes:
1. Fix the issues listed in review-feedback.md
2. Run: /jira-to-pr-workflow {ticket_id}
3. Workflow will resume from checkpoint
4. Reviews will be re-run automatically
""")
        return False  # Exit skill (resume later)
```

## Agent-Specific Invocation Patterns

### code-reviewer Agent

**Specialization:** General code review (architecture, patterns, security)

**Invocation:**
```
Skill: code-reviewer
Args: {changed_files}

Focus:
- Architecture and design patterns
- SOLID principles
- Security vulnerabilities (XSS, injection, etc.)
- Error handling
- Code organization
```

### swift-expert Agent

**Specialization:** Swift language deep review

**Invocation:**
```
Skill: swift-expert
Args: {changed_files}

Focus:
- Swift 6.0+ modern idioms
- Async/await patterns
- Actor isolation
- Memory management (ARC, weak/unowned)
- Protocol-oriented design
- Value types vs reference types
```

### swiftui-specialist Agent

**Specialization:** SwiftUI architecture and performance

**Invocation:**
```
Skill: swiftui-specialist
Args: {changed_files}

Focus:
- View composition and hierarchy
- State management (@State, @Binding, @Observable)
- Performance (view body complexity, unnecessary re-renders)
- SwiftUI best practices
- Accessibility
- Layout and animation
```

### swift-reviewer Agent

**Specialization:** Swift best practices and conventions

**Invocation:**
```
Skill: swift-reviewer
Args: {changed_files}

Focus:
- Swift API design guidelines
- Naming conventions
- Code style consistency
- Optional handling (guard, if-let, nil coalescing)
- Error types and throwing functions
- Documentation comments
```
