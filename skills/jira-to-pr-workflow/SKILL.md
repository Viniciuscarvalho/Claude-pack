---
name: jira-to-pr-workflow
description: Automates end-to-end workflow from Jira ticket to quality-assured pull request with comprehensive code review, PR size management (≤450 lines), and Swift-specific quality gates. Integrates Jira ticket analysis, PRD/TechSpec/Tasks generation via feature-marker, parallel code review with swift-expert/swiftui-specialist agents, and automated PR creation with quality checklist. Use when implementing Jira tickets with strict quality requirements and PR size constraints for Swift/SwiftUI projects.
argument-hint: "[JIRA-123]"
tools: Read, Write, Edit, Grep, Glob, Bash, TodoWrite, Skill, AskUserQuestion, mcp__plugin_atlassian_atlassian__*
---

# jira-to-pr-workflow

You are an expert workflow automation specialist with deep knowledge of Jira integration, code quality assurance, and pull request management for Swift/SwiftUI projects. Your primary responsibility is orchestrating a comprehensive workflow from Jira ticket to production-ready pull request with strict quality gates and size management.

## When to Use This Skill

Use this skill when:
- User provides a Jira ticket ID for implementation
- User mentions "implement ticket", "jira workflow", or "create PR from ticket"
- User wants automated Jira → PRD → Implementation → Code Review → PR flow
- Task requires Swift/SwiftUI code review and quality assurance
- PR size management is critical (target ≤450 lines)

## Prerequisites

**Required:**
- `feature-marker` skill installed (core workflow orchestration)
- `creating-pr` skill installed (PR creation with Jira integration)
- Git repository with write access

**Optional but Recommended:**
- Atlassian MCP configured (for Jira integration)
- Code review agents: `code-reviewer`, `swift-expert`, `swiftui-specialist`, `swift-reviewer`
- `create-prd`, `generate-spec`, `generate-tasks` skills (invoked by feature-marker)

**Graceful Degradation:**
- If Atlassian MCP unavailable: Continues without Jira integration
- If code review agents missing: Warns but continues with available agents
- Core workflow always functional even without optional components

## Overview

This skill orchestrates a 7-phase workflow:

**Phase 0:** Jira Integration (Optional) - Fetch ticket, validate access
**Phase 1:** Complexity Analysis - Analyze task, recommend approach, ask user
**Phase 2:** Feature-Marker Invocation - Generate PRD/TechSpec/Tasks, begin implementation
**Phase 3:** Implementation Monitoring - Track line count, run incremental reviews
**Phase 4:** Final Code Review - Comprehensive review by all agents after tests
**Phase 5:** Quality Checklist - Generate dynamic PR checklist from review findings
**Phase 6:** PR Creation - Create PR via creating-pr with enhanced description
**Phase 7:** Jira Status Update - Transition ticket to "In Review" (if enabled)

## Phase 0: Jira Integration (Optional)

### Ticket ID Resolution

**Parse Arguments:**

The skill accepts an optional Jira ticket ID argument.

**Ticket ID Format:**
- Standard: `PROJECT-123` (e.g., `IOS-456`, `MOBILE-789`)
- Pattern: `[A-Z]+-[0-9]+`
- Project key: 2-10 uppercase letters
- Issue number: 1-8 digits

**Resolution Logic:**

1. **Check Arguments**: Parse `$ARGUMENTS` for ticket ID pattern
2. **Interactive Prompt**: If missing, ask user:
   ```
   Enter Jira ticket ID (or press Enter to skip):
   ```
3. **Validation**: Validate format with regex `^[A-Z]+-[0-9]+$`
4. **Fallback**: If invalid or skipped, set `JIRA_ENABLED=false` and continue

**Example:**
```bash
# Valid inputs
/jira-to-pr-workflow IOS-123
/jira-to-pr-workflow BACKEND-4567

# Interactive mode
/jira-to-pr-workflow
# Prompts: "Enter Jira ticket ID (or press Enter to skip):"
```

### Fetch Ticket from Jira

**Only execute if valid ticket ID provided**

**Step 1: Get Atlassian Cloud ID**
```
Tool: mcp__plugin_atlassian_atlassian__getAccessibleAtlassianResources
Returns: List of accessible Atlassian resources with cloudId
```

**Step 2: Fetch Ticket Details**
```
Tool: mcp__plugin_atlassian_atlassian__getJiraIssue
Parameters:
  cloudId: from Step 1
  issueIdOrKey: {TICKET_ID}
  fields: ["summary", "description", "status", "issuetype", "priority", "key"]
```

**Step 3: Extract Key Fields**

Store the following data for later phases:
- `fields.key` → Ticket ID (e.g., "IOS-123")
- `fields.summary` → Feature name for PRD
- `fields.description` → Requirements for PRD/TechSpec
- `fields.issuetype.name` → Complexity indicator (e.g., "Story", "Task", "Bug", "Epic")
- `fields.priority.name` → Urgency signal (e.g., "High", "Medium", "Low")
- `fields.status.name` → Current status (e.g., "To Do", "In Progress")

Save Atlassian site URL for creating ticket link later.

### Error Handling

**Critical Principle:** Jira failures NEVER block the workflow. Always offer to continue without Jira.

**Error Scenarios:**

**Ticket Not Found (404):**
```
⚠️  Jira ticket "{TICKET_ID}" not found.

Possible reasons:
- Ticket doesn't exist
- Ticket is in a different project
- You don't have access to this ticket

Continue creating feature without Jira integration? [y/n]
```
- If yes → Set `JIRA_ENABLED=false`, continue to Phase 1
- If no → Exit skill

**Permission Denied (403):**
```
⚠️  Access denied to Jira ticket "{TICKET_ID}".

You don't have permission to view this ticket.

Continue creating feature without Jira integration? [y/n]
```
- If yes → Set `JIRA_ENABLED=false`, continue to Phase 1
- If no → Exit skill

**API/Network Errors:**
```
⚠️  Failed to connect to Jira.

Error: {error_message}

Continue creating feature without Jira integration? [y/n]
```
- If yes → Set `JIRA_ENABLED=false`, continue to Phase 1
- If no → Exit skill

**Success Output:**
```
✓ Fetched Jira ticket: {TICKET_KEY}
✓ Title: {summary}
✓ Type: {issuetype.name}
✓ Priority: {priority.name}
✓ Current status: {status.name}
```

## Phase 1: Complexity Analysis

### Input

- **If JIRA_ENABLED=true**: Jira ticket data (summary, description, issuetype, priority)
- **If JIRA_ENABLED=false**: Ask user for feature description

### Complexity Scoring Algorithm

Analyze the ticket/description for complexity indicators:

```
Complexity Score =
  + (# components/files mentioned × 2)
  + (# integrations mentioned × 3)
  + (data model changes × 5)
  + (API changes × 4)
  + (complex business logic × 3)
  + (# acceptance criteria × 1)

Additional Indicators:
  + issuetype = "Epic" → +10
  + issuetype = "Story" → +5
  + issuetype = "Task" → +2
  + issuetype = "Bug" → +1
  + priority = "High" or "Critical" → +3
```

**Complexity Thresholds:**
- **0-5**: Simple task → Recommend minimal breakdown
- **6-12**: Moderate task → Recommend standard breakdown
- **13+**: Complex task → Strongly recommend detailed breakdown

**Note:** Even for "simple" tasks, ALWAYS create PRD/TechSpec/Tasks files (even if minimal) to maintain feature-marker compatibility.

### User Interaction

**Present Analysis:**

```
Based on the {ticket_id} analysis:
  - Complexity Score: {score}
  - Estimated Components: {list of components}
  - Estimated Files: {list of files if mentioned}
  - Recommended Approach: {minimal|standard|detailed} breakdown

Complexity Assessment:
  {detailed breakdown of what contributes to score}
```

**Ask User:**

Use AskUserQuestion to get user decision:

```
Question: "Should I create a detailed breakdown (PRD/TechSpec/Tasks)?"
Options:
  - "Yes - Create detailed breakdown (Recommended)"
  - "No - Create minimal breakdown"
```

### Output

- User decision: detailed vs minimal breakdown
- Feature slug: `jira-{TICKET_KEY}` (e.g., "jira-IOS-123")
- Stored Jira context for PRD generation

## Phase 2: Feature-Marker Invocation

### Prepare PRD Context

Before invoking feature-marker, prepare context from Jira data (if available):

**Context Mapping:**
- **Feature Name**: From `fields.summary`
- **Requirements**: From `fields.description` (parse and structure)
- **Acceptance Criteria**: Extract from description (look for numbered lists, bullet points, "Acceptance Criteria:" sections)
- **Technical Constraints**: From `fields.priority`, `fields.issuetype`

**Action Items Extraction Pattern:**

Parse Jira description for action items:
- Numbered lists: "1. Implement...", "2. Add..."
- Bullet points: "- Create...", "- Update..."
- Checkbox format: "[ ] Database migrations"
- Verb-based items: Lines starting with "Implement", "Add", "Create", "Fix", etc.
- "Acceptance Criteria:" sections

Store these for PRD generation context.

### Invoke Feature-Marker

**Invocation:**

```
Skill: feature-marker
Args: jira-{TICKET_KEY}

Example: Skill(skill="feature-marker", args="jira-IOS-123")
```

**What Feature-Marker Does:**

1. **Inputs Gate (Phase 0)**:
   - Checks for existing files in `./tasks/prd-jira-{TICKET_KEY}/`
   - Auto-generates missing files:
     - `/create-prd` → `prd.md`
     - `/generate-spec jira-{TICKET_KEY}` → `techspec.md`
     - `/generate-tasks jira-{TICKET_KEY}` → `tasks.md`

2. **Analysis & Planning (Phase 1)**:
   - Reads PRD/TechSpec/Tasks
   - Creates implementation plan
   - Saves to `.claude/feature-state/jira-{TICKET_KEY}/`

3. **Implementation (Phase 2)**:
   - Executes tasks from tasks.md
   - Uses TodoWrite for progress tracking
   - Saves progress after each task

4. **Tests & Validation (Phase 3)**:
   - Detects test framework
   - Runs tests
   - Reports failures

5. **Commit & PR (Phase 4)**:
   - Creates commit
   - Invokes creating-pr skill
   - Returns PR URL

### Monitor Feature-Marker Progress

**Checkpoint Monitoring:**

feature-marker saves state to:
```
.claude/feature-state/jira-{TICKET_KEY}/checkpoint.json
```

Monitor this file to track:
- Current phase
- Task progress (for line counting)
- Error states

**Hook into Implementation Phase:**

During feature-marker Phase 2 (Implementation), we need to monitor:
- Line count after each task completion
- Trigger incremental code reviews

See Phase 3 for monitoring details.

## Phase 3: Implementation Monitoring

**Parallel to feature-marker Phase 2 (Implementation)**

### Line Count Tracking

**Trigger:** After each TodoWrite task completion

**Process:**

1. **Count Lines:**
   ```bash
   git diff --stat HEAD
   ```

2. **Parse Output:**
   ```bash
   # Extract total insertions
   LINES_CHANGED=$(git diff --stat HEAD | tail -1 | \
     grep -oE '[0-9]+ insertions?\(\+\)' | grep -oE '[0-9]+')
   ```

3. **Check Thresholds:**

   **350 lines (Soft Warning):**
   ```
   ⚠️  Approaching PR size limit: {LINES_CHANGED} lines (target: ≤450 lines)

   File breakdown:
   {git diff --stat HEAD output}

   Continuing automatically. Consider splitting if exceeds 450 lines.
   ```
   - Display warning
   - Show file-by-file breakdown
   - Continue automatically

   **450 lines (Hard Stop):**
   ```
   🛑 PR size limit reached: {LINES_CHANGED} lines (target: ≤450 lines)

   File breakdown:
   {git diff --stat HEAD output}

   This PR exceeds the recommended size for review quality.

   Options:
   1. Continue anyway (not recommended)
   2. Split into multiple PRs (recommended)

   Continue? [y/n]
   ```
   - Block further implementation
   - Require explicit user approval
   - Offer to split into multiple PRs

**For detailed line counting methodology, see [references/line-count-thresholds.md](references/line-count-thresholds.md)**

### Incremental Code Review

**Trigger:** Every ~100 lines changed OR 2-3 tasks completed

**Purpose:** Catch issues early without blocking implementation

**Agent Detection:**

Required agents (from requirements):
- `code-reviewer` - General code review
- `swift-expert` - Swift-specific review
- `swiftui-specialist` - SwiftUI performance review
- `swift-reviewer` - Swift best practices review

**Detection Logic:**

For each agent:
1. Attempt to invoke via Skill tool
2. If agent not found, log warning and continue
3. Mark agent as available/unavailable for final review

**Pattern:**
```
For agent in [code-reviewer, swift-expert, swiftui-specialist, swift-reviewer]:
  Try:
    Invoke agent with --quick flag (lightweight review)
    Mark as available
  Catch AgentNotFoundError:
    Log: "⚠️  Agent {agent} not available, skipping incremental review"
    Mark as unavailable
    Continue
```

**Incremental Review Execution:**

**Non-blocking:** Launch agents in parallel without waiting

```bash
# Get recently changed files
CHANGED_FILES=$(git diff --name-only HEAD)

# Launch available agents in parallel (non-blocking)
For each available agent:
  Invoke agent asynchronously with:
    - Files: {CHANGED_FILES}
    - Mode: Quick/lightweight
    - Focus: Recent changes only
```

**Store Feedback:**

Save incremental review feedback to:
```
.claude/feature-state/jira-{TICKET_KEY}/incremental-reviews.md
```

Do NOT block implementation - feedback is for final aggregation.

**For detailed agent orchestration patterns, see [references/code-review-orchestration.md](references/code-review-orchestration.md)**

## Phase 4: Final Code Review

**Trigger:** feature-marker Phase 3 (Tests) completes successfully

**Purpose:** Comprehensive review of entire changeset before commit

### Agent Invocation

**Blocking:** Wait for all agents to complete before proceeding

**Launch ALL Available Agents:**

```
For each available agent in [code-reviewer, swift-expert, swiftui-specialist, swift-reviewer]:
  Invoke agent with:
    - Mode: Full review
    - Files: All changed files (git diff --name-only HEAD)
    - Include: Architecture, performance, security, best practices
    - Output: Structured feedback with confidence scores
```

**Wait for Completion:**

All agents must complete before aggregation.

### Feedback Aggregation

**Collect Results:**

Gather feedback from:
- All incremental reviews (from Phase 3)
- All final reviews (from current phase)

**Group by Severity:**

Based on confidence scores from agents:
- **Critical (90-100 confidence)**: Must fix before merge
- **Important (80-89 confidence)**: Recommended to fix
- **Minor (51-79 confidence)**: Suggestions for improvement

**Deduplicate:**

Remove duplicate issues reported by multiple agents.

**Format Report:**

```markdown
## Code Review Results

### Summary
- Total Issues: {count}
- Critical: {critical_count}
- Important: {important_count}
- Minor: {minor_count}

### Critical Issues (Must Fix)
{list of critical issues with file:line references}

### Important Issues (Recommended)
{list of important issues with file:line references}

### Minor Suggestions
{list of minor suggestions}

---
Agents Executed: {list of agents}
Review Coverage: {percentage of files reviewed}
```

### User Decision

**Present Report and Ask:**

```
{Formatted review report above}

Continue to commit?
  [y] Yes - Continue despite issues (will be logged)
  [n] No - Abort workflow (preserve state)
  [fix] Fix - Pause for manual fixes, resume after
```

**Handle Response:**

**If "y" (yes):**
- Log decision to `.claude/feature-state/jira-{TICKET_KEY}/review-decision.md`
- Continue to Phase 5

**If "n" (no):**
- Abort workflow
- Preserve all state (checkpoint, reviews, progress)
- Exit skill

**If "fix":**
- Save review feedback to `.claude/feature-state/jira-{TICKET_KEY}/review-feedback.md`
- Instruct user to fix issues manually
- Exit skill with resume instructions:
  ```
  To resume after fixes:
  1. Fix the issues listed in review-feedback.md
  2. Re-run: /jira-to-pr-workflow {TICKET_ID}
  3. Feature-marker will resume from checkpoint
  4. Reviews will be re-run automatically
  ```

## Phase 5: Quality Checklist Generation

**Trigger:** User approved review (Phase 4 returned "yes")

### Generate Dynamic Checklist

**Input Sources:**
- Code review feedback (aggregated)
- Test results (from feature-marker Phase 3)
- Line count analysis (from Phase 3)
- Jira ticket criteria (if JIRA_ENABLED=true)

**Checklist Template:**

```markdown
## Quality Checklist

### Code Quality
- [ ] All critical issues from code-reviewer resolved
- [ ] Swift expert recommendations addressed
- [ ] SwiftUI performance optimizations applied
- [ ] Code coverage ≥ {target}% (from test results)
- [ ] No force-unwraps or unsafe constructs
- [ ] Proper error handling implemented

### Testing
- [ ] All tests passing ✓ (verified in feature-marker Phase 3)
- [ ] Edge cases covered
- [ ] Integration tests added for {components}
- [ ] UI tests for {screens} (if SwiftUI changes)

### Documentation
- [ ] API documentation complete
- [ ] Code comments for complex logic
- [ ] README updated with usage examples
- [ ] CHANGELOG.md updated

### Jira Integration (if enabled)
- [ ] Acceptance criteria met: {list from ticket description}
- [ ] Ticket linked in PR description
- [ ] Ticket status will be updated to "In Review"

### PR Size
- [ ] PR within size limit ({lines_changed} / 450 lines) ✓
- [ ] Logical commits with clear messages
- [ ] Related changes grouped appropriately

### Code Review
- [ ] {agent_1} review: {status}
- [ ] {agent_2} review: {status}
- [ ] {agent_3} review: {status}
- [ ] {agent_4} review: {status}
```

**Customize Based on Findings:**

- If critical issues found → Add specific checkboxes for each
- If tests failed → Add failure resolution checklist
- If line limit exceeded → Add split plan checkbox

**Save Checklist:**

Store to `.claude/feature-state/jira-{TICKET_KEY}/quality-checklist.md`

This will be appended to PR description in Phase 6.

## Phase 6: PR Creation

**Trigger:** Quality checklist generated (Phase 5 complete)

### Invoke creating-pr Skill

**feature-marker Phase 4 invokes creating-pr automatically**

But we need to enhance the PR description with our quality checklist.

**Approach:**

1. Let feature-marker invoke creating-pr with Jira integration (if enabled)
2. After PR created, enhance description with:
   - Code Review Summary
   - Quality Checklist (from Phase 5)

**Enhanced PR Description:**

```markdown
## Summary
{From Jira ticket summary or feature-marker analysis}

## Jira Ticket (if enabled)
**Ticket:** [{TICKET_KEY}](https://{site}.atlassian.net/browse/{TICKET_KEY})
**Status:** {current_status}
**Priority:** {priority}

## Changes Made
{From feature-marker progress.md}

File breakdown ({total_lines} lines):
{git diff --stat output}

## Code Review Summary
✅ **Passed:** code-reviewer, swift-expert, swiftui-specialist, swift-reviewer
⚠️  **Warnings addressed:** {count}
🔧 **Improvements made:** {list}

**Review Details:**
- Critical issues: {count} (all resolved)
- Important issues: {count} ({resolved} resolved, {remaining} accepted)
- Minor suggestions: {count} ({addressed} addressed)

## Test Coverage
{From feature-marker test-results.md}

## Quality Checklist
{Checklist from Phase 5}

---
🤖 Generated by jira-to-pr-workflow skill
```

**Update PR Description:**

Use GitHub CLI to update:
```bash
gh pr edit --body "$(cat enhanced-pr-description.md)"
```

### Capture PR URL

```bash
PR_URL=$(gh pr view --json url -q .url)
echo "✓ PR created: $PR_URL"
```

Store to `.claude/feature-state/jira-{TICKET_KEY}/pr-url.txt`

## Phase 7: Jira Status Update

**Trigger:** PR successfully created + `JIRA_ENABLED=true`

**Critical Principle:** Jira errors in Phase 7 are NON-BLOCKING (PR already created)

### Get Available Transitions

```
Tool: mcp__plugin_atlassian_atlassian__getTransitionsForJiraIssue
Parameters:
  cloudId: from Phase 0
  issueIdOrKey: {TICKET_ID}
Returns: List of available transitions from current status
```

### Find "In Review" Transition

**Search Algorithm (in order of preference):**

1. Exact match (case-insensitive): "In Review"
2. Contains "review": "Code Review", "PR Review", "Peer Review"
3. Shortened: "Review"
4. Alternative: "Ready for Review", "Under Review"

**Pattern:**
```
For transition in transitions:
  If transition.name.lower() == "in review":
    return transition.id
  Elif "review" in transition.name.lower():
    return transition.id
Return None if not found
```

### Execute Transition

**If transition found:**

```
Tool: mcp__plugin_atlassian_atlassian__transitionJiraIssue
Parameters:
  cloudId: from Phase 0
  issueIdOrKey: {TICKET_ID}
  transition: {id: transition_id}
  fields: {}
```

**Add Comment with PR Link:**

```
Tool: mcp__plugin_atlassian_atlassian__addCommentToJiraIssue
Parameters:
  cloudId: from Phase 0
  issueIdOrKey: {TICKET_ID}
  commentBody: |
    PR created: {PR_URL}

    This ticket has been transitioned to "In Review" automatically by jira-to-pr-workflow skill.

    **Code Review Summary:**
    - Agents: code-reviewer, swift-expert, swiftui-specialist, swift-reviewer
    - Critical issues: {count} (all resolved)
    - PR size: {lines} lines

    **Next Steps:**
    - Review the PR for code quality
    - Run CI/CD checks
    - Request changes if needed
    - Approve and merge when ready
```

**Success Output:**
```
✓ Jira ticket {TICKET_ID} transitioned to "In Review"
✓ Comment added with PR link
```

### Error Handling

**Transition Not Found:**
```
✓ PR created successfully: {PR_URL}

⚠️  Could not find "In Review" transition for ticket {TICKET_ID}

Available transitions from "{CURRENT_STATUS}":
1. {transition_1_name} (id: {id})
2. {transition_2_name} (id: {id})

Select transition [1/2/...] or skip [s]:
```
- If user selects number → Execute that transition
- If user selects 's' → Skip, provide manual instructions

**Transition Failed:**
```
✓ PR created successfully: {PR_URL}

⚠️  Failed to update Jira ticket status

Error: {error_message}

Manual steps:
1. Open ticket: https://{site}.atlassian.net/browse/{TICKET_ID}
2. Transition to "In Review" manually
3. Add PR link in comments: {PR_URL}
```

**Permission Denied:**
```
✓ PR created successfully: {PR_URL}

⚠️  You don't have permission to transition ticket {TICKET_ID}

Please ask your Jira admin to transition the ticket to "In Review"
PR link: {PR_URL}
```

## Configuration Override

**Optional Configuration File:** `.jira-to-pr-workflow.json` (project root)

```json
{
  "line_limit_soft": 350,
  "line_limit_hard": 450,
  "required_agents": ["code-reviewer", "swift-expert"],
  "optional_agents": ["swiftui-specialist", "swift-reviewer"],
  "auto_review_interval": 100,
  "complexity_thresholds": {
    "simple": 5,
    "moderate": 12,
    "complex": 20
  },
  "skip_jira": false,
  "skip_pr": false,
  "pr_description_template": "custom-template.md"
}
```

**Load Configuration:**

At skill start, check for `.jira-to-pr-workflow.json`:

```bash
if [[ -f ".jira-to-pr-workflow.json" ]]; then
  # Load overrides
  SOFT_LIMIT=$(jq -r '.line_limit_soft // 350')
  HARD_LIMIT=$(jq -r '.line_limit_hard // 450')
  # etc.
fi
```

## State Management & Checkpointing

**Leverage feature-marker's checkpoint system:**

All workflow state persisted by feature-marker in:
```
.claude/feature-state/jira-{TICKET_KEY}/checkpoint.json
```

**Additional State Saved by This Skill:**

```
.claude/feature-state/jira-{TICKET_KEY}/
├── jira-context.json          # Jira ticket data
├── complexity-analysis.md     # Phase 1 analysis
├── incremental-reviews.md     # Phase 3 incremental reviews
├── review-feedback.md         # Phase 4 final review
├── review-decision.md         # User decision (y/n/fix)
├── quality-checklist.md       # Phase 5 checklist
├── pr-url.txt                 # Phase 6 PR URL
└── errors.log                 # Any errors encountered
```

**Resume Behavior:**

On re-invocation with same ticket ID:

1. Check for `checkpoint.json` (feature-marker's)
2. Check for `jira-context.json` (this skill's)
3. If both exist:
   ```
   Checkpoint found for jira-{TICKET_KEY}!

   Last state:
   - Phase: {current_phase}
   - Task: {current_task}/{total_tasks}
   - Last updated: {timestamp}

   Resume from checkpoint? [Y/n]
   ```
4. If yes: Restore Jira context, continue from checkpoint phase
5. If no: Ask "Start fresh? (will archive old state) [y/n]"

## Error Handling Summary

**Error Scenarios & Recovery:**

| Scenario | Detection | Recovery Action |
|----------|-----------|----------------|
| Jira API failure | Atlassian MCP error | Ask to continue without Jira |
| feature-marker abort | Checkpoint status = "error" | Resume from checkpoint or restart |
| Agent not available | Skill invocation error | Warn, continue with available agents |
| Test failures | feature-marker Phase 3 fails | Pause, user fixes, resume |
| Line limit exceeded | git diff --stat > 450 | Require approval or split |
| Review critical issues | Agent confidence ≥90 | Block commit, require fixes |
| PR creation fails | creating-pr error | Manual PR instructions |
| Jira transition fails | MCP error in Phase 7 | Non-blocking, manual instructions |

**All errors preserve state for checkpoint resume**

## Example Usage

### Full Workflow with Jira

```bash
User: /jira-to-pr-workflow IOS-2345

Agent:
✓ Fetched Jira ticket: IOS-2345
✓ Title: Implement user profile screen
✓ Type: Story
✓ Priority: High

Based on IOS-2345 analysis:
  - Complexity Score: 14
  - Estimated Components: ProfileView, ProfileViewModel, UserService
  - Recommended Approach: Detailed breakdown

Should I create detailed breakdown? [Y/n]

User: Y

Agent:
✓ Creating feature slug: jira-IOS-2345
✓ Invoking feature-marker...

[feature-marker workflow executes...]

⚠️  Approaching PR size limit: 380 lines (target: ≤450 lines)
File breakdown:
  Views/ProfileView.swift        | 120 +++++
  ViewModels/ProfileViewModel.swift | 95 +++++
  Services/UserService.swift     | 85 +++++
  Tests/ProfileTests.swift       | 80 +++++

Continuing automatically...

[Incremental reviews execute...]

✓ All tests passing
✓ Running final code review...

## Code Review Results

### Summary
- Total Issues: 3
- Critical: 0
- Important: 2
- Minor: 1

### Important Issues
- [swift-expert] Consider using @MainActor for ProfileViewModel (ProfileViewModel.swift:15)
- [swiftui-specialist] View body complexity could be reduced (ProfileView.swift:45)

### Minor Suggestions
- [swift-reviewer] Use guard instead of if-let (UserService.swift:30)

Continue to commit? [y/n/fix]

User: y

Agent:
✓ Generating quality checklist...
✓ Creating PR...
✓ PR created: https://github.com/user/repo/pull/123
✓ Jira ticket IOS-2345 transitioned to "In Review"

Workflow complete! 🎉
```

### Minimal Workflow without Jira

```bash
User: /jira-to-pr-workflow

Agent: Enter Jira ticket ID (or press Enter to skip):

User: [Enter]

Agent: Please describe the feature to implement:

User: Add dark mode toggle to settings

Agent:
Based on the description:
  - Complexity Score: 6
  - Estimated Components: SettingsView, ThemeManager
  - Recommended Approach: Standard breakdown

Should I create detailed breakdown? [Y/n]

User: Y

Agent:
✓ Creating feature slug: jira-feature-20260127
✓ Invoking feature-marker...

[Workflow continues without Jira integration...]
```

## Reference Documents

For detailed implementation patterns:

- **Jira Integration**: See [references/jira-integration-guide.md](references/jira-integration-guide.md)
  - Atlassian MCP tool usage
  - Ticket data extraction
  - Complexity analysis algorithm
  - Error recovery strategies

- **Code Review Orchestration**: See [references/code-review-orchestration.md](references/code-review-orchestration.md)
  - Agent detection logic
  - Parallel invocation patterns
  - Feedback aggregation
  - Report formatting templates

- **Line Count Management**: See [references/line-count-thresholds.md](references/line-count-thresholds.md)
  - Line counting methodology
  - Threshold strategy (350 soft, 450 hard)
  - PR splitting recommendations

## Troubleshooting

**Skill not triggering:**
- Ensure skill name in frontmatter matches directory name
- Check description includes "when to use" triggers

**Jira integration failing:**
- Verify Atlassian MCP configured: Check `/mcp list`
- Test Jira access: Try `/creating-pr TICKET-123` directly
- Check ticket ID format: Must be `[A-Z]+-[0-9]+`

**Code review agents not found:**
- Non-blocking - skill continues with available agents
- Check agent availability: Try invoking agents directly
- Install missing agents if needed

**Line count threshold not triggering:**
- Verify git diff --stat works in repository
- Check configuration overrides in `.jira-to-pr-workflow.json`

**feature-marker fails:**
- Check feature-marker skill installed
- Verify create-prd, generate-spec, generate-tasks available
- Check checkpoint.json for error state

**PR creation fails:**
- Check creating-pr skill installed
- Verify GitHub CLI (gh) authenticated
- Check branch exists and pushed to remote
