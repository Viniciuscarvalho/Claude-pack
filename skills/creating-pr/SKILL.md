---
name: creating-pr
description: Use when creating or updating pull requests with comprehensive descriptions and meaningful commits. Optionally integrates with Jira to populate PR content from ticket details and update ticket status - streamlines PR workflow with branch management, commit best practices, and Jira synchronization
argument-hint: "[TICKET-ID]"
---

You are an expert Git and GitHub workflow automation specialist with deep knowledge of version control best practices and pull request management. Your primary responsibility is streamlining the pull request creation process, ensuring high-quality commits with meaningful descriptions.

## Jira Integration (Optional)

When invoked with a Jira ticket ID (e.g., `/creating-pr PROJ-123`), this skill will:

1. Fetch ticket details from Jira using the Atlassian MCP
2. Use ticket `summary` as PR title
3. Use ticket `description` as PR body content (converting action points to checklist)
4. Automatically transition ticket to "In Review" status after PR creation

**Usage:**
- **With Jira:** `/creating-pr PROJ-123`
- **Without Jira:** `/creating-pr` (uses standard workflow)

**Benefits:**
- Eliminates manual copy-paste from Jira to PR
- Ensures PR description matches ticket requirements
- Keeps Jira status in sync with development workflow
- Maintains full traceability between code changes and tickets

**Requirements:**
- Atlassian MCP server must be configured
- User must have read access to the Jira ticket
- User must have permission to transition ticket status

## Parameter Parsing

The skill accepts an optional ticket ID argument.

**Ticket ID Format:**
- Standard: `PROJECT-123` (e.g., `BACKEND-456`, `MOB-789`)
- Pattern: `[A-Z]+-[0-9]+`
- Project key: 2-10 uppercase letters
- Issue number: 1-8 digits

**Examples:**
- Valid: `PROJ-123`, `BACKEND-12345`, `AB-1`
- Invalid: `proj-123` (lowercase), `PROJ123` (missing hyphen), `PROJ-` (missing number)

**Validation Logic:**
```bash
# Parse and validate ticket ID
TICKET_ID="$ARGUMENTS"

if [[ -n "$TICKET_ID" && "$TICKET_ID" =~ ^[A-Z]+-[0-9]+$ ]]; then
  echo "✓ Valid ticket ID: $TICKET_ID"
  JIRA_ENABLED=true
else
  if [[ -n "$TICKET_ID" ]]; then
    echo "⚠ Invalid ticket ID format: $TICKET_ID"
  fi
  echo "Using standard workflow without Jira integration"
  JIRA_ENABLED=false
fi
```

## Common Operations

### GitHub CLI Commands Reference

```bash
# PR Management
gh pr view                                    # View current branch PR
gh pr list                                    # List open PRs
gh pr view <number> --json number -q .number # Get PR number
gh pr create --title "" --body ""            # Create new PR
gh pr edit --body ""                         # Update description
gh pr edit --add-label ""                    # Add labels

# Git Commands
git branch --show-current                    # Current branch
git status                                   # Check changes
git diff                                     # View unstaged changes
git diff --cached                           # View staged changes
git diff HEAD~1..HEAD                       # Last commit diff
git rev-parse HEAD                          # Get commit SHA
git log -1 --pretty=%s                      # Last commit message
```

## Workflow

### Creating/Updating Pull Requests

0. **Parse Arguments & Fetch Jira (Optional)**:

   **Only execute if ticket ID provided in arguments**

   Parse and validate ticket ID:
   ```bash
   TICKET_ID="$ARGUMENTS"

   if [[ -n "$TICKET_ID" && "$TICKET_ID" =~ ^[A-Z]+-[0-9]+$ ]]; then
     JIRA_ENABLED=true
   else
     JIRA_ENABLED=false
     # Skip to Step 1 if no valid ticket ID
   fi
   ```

   If `JIRA_ENABLED=true`, fetch ticket from Jira:

   **Get Atlassian Cloud ID:**
   ```bash
   # Use mcp__plugin_atlassian_atlassian__getAccessibleAtlassianResources
   # Returns list of accessible Atlassian resources with cloudId
   ```

   **Fetch Ticket Details:**
   ```bash
   # Use mcp__plugin_atlassian_atlassian__getJiraIssue
   # Parameters:
   #   cloudId: from getAccessibleAtlassianResources
   #   issueIdOrKey: $TICKET_ID
   #   fields: ["summary", "description", "status", "key"]
   ```

   **Extract Key Fields:**
   - `fields.key` → Ticket ID (e.g., "PROJ-123")
   - `fields.summary` → Will become PR title
   - `fields.description` → Will become PR body content
   - `fields.status.name` → Current status (e.g., "To Do", "In Progress")
   - Save Atlassian site URL for creating ticket link

   **Error Handling:**

   - **Ticket Not Found (404):**
     ```
     ⚠️ Jira ticket "{TICKET_ID}" not found.

     Possible reasons:
     - Ticket doesn't exist
     - Ticket is in a different project
     - You don't have access to this ticket

     Continue creating PR without Jira integration? [y/n]
     ```
     If yes → Set `JIRA_ENABLED=false`, continue to Step 1
     If no → Exit skill

   - **Permission Denied (403):**
     ```
     ⚠️ Access denied to Jira ticket "{TICKET_ID}".

     Continue creating PR without Jira integration? [y/n]
     ```
     If yes → Set `JIRA_ENABLED=false`, continue to Step 1
     If no → Exit skill

   - **API/Network Errors:**
     ```
     ⚠️ Failed to connect to Jira.

     Error: {error_message}

     Continue creating PR without Jira integration? [y/n]
     ```
     If yes → Set `JIRA_ENABLED=false`, continue to Step 1
     If no → Exit skill

   **Success Output:**
   ```
   ✓ Fetched Jira ticket: {TICKET_KEY}
   ✓ Title: {summary}
   ✓ Current status: {status.name}
   ```

1. **Branch Management**:

   - Check current branch: `git branch --show-current`
   - If on main/master/next, create feature branch with conventional naming
   - Switch to new branch: `git checkout -b branch-name`

2. **Analyze & Stage**:

   - Review changes: `git status` and `git diff`
   - Identify change type (feature, fix, refactor, docs, test, chore)
   - Stage ALL changes: `git add .` (preferred due to slow Husky hooks)
   - Verify: `git diff --cached`

3. **Commit & Push**:

   - **Single Commit Strategy**: Use one comprehensive commit per push due to slow Husky hooks
   - Format: `type: brief description` (simple format preferred)
   - Commit: `git commit -m "type: description"` with average git comment
   - Push: `git push -u origin branch-name`

4. **PR Management**:

   - Check existing: `gh pr view`
   - If exists: push updates, **add update comment** (preserve original description)
   - If not: Create new PR with Jira-enhanced content (if available) or standard format

   **Create PR Title:**

   - **With Jira (`JIRA_ENABLED=true`):**
     ```bash
     PR_TITLE="$JIRA_SUMMARY"
     ```

   - **Without Jira:**
     ```bash
     # Use conventional format from commit message
     COMMIT_MSG=$(git log -1 --pretty=%s)
     PR_TITLE="$COMMIT_MSG"
     ```

   **Create PR Body:**

   - **With Jira (`JIRA_ENABLED=true`):**

     Extract action points from Jira description (see "Action Points Extraction" section):
     ```bash
     # Parse $JIRA_DESCRIPTION to separate:
     # - Summary content (non-actionable text)
     # - Action items (requirements, acceptance criteria, numbered/bulleted lists)
     ```

     Format PR body:
     ```markdown
     ## Summary

     {Summary content from Jira description - non-action items}

     ## Jira Ticket

     **Ticket:** [{JIRA_KEY}](https://{ATLASSIAN_SITE}.atlassian.net/browse/{JIRA_KEY})
     **Current Status:** {JIRA_STATUS}

     ## Test Plan

     {Action items from Jira description converted to checklist}
     - [ ] Action item 1
     - [ ] Action item 2
     - [ ] Action item 3

     ## Changes Made

     {Git diff summary}

     ```

   - **Without Jira:**
     ```markdown
     ## Summary

     {Generated from commit analysis}

     ## Changes Made

     {Git diff summary}

     ## Test Plan

     - [ ] {Generated testing checklist based on changes}

     ```

   **Create PR:**
   ```bash
   gh pr create --title "$PR_TITLE" --body "$(cat <<'EOF'
   {PR_BODY from above}
   EOF
   )"
   ```

   **Capture PR URL:**
   ```bash
   PR_URL=$(gh pr view --json url -q .url)
   echo "✓ PR created: $PR_URL"
   ```

5. **Update Jira Status (If Jira Integration Active)**:

   **Only execute if:**
   - `JIRA_ENABLED=true`
   - Jira ticket was successfully fetched in Step 0
   - PR was successfully created in Step 4

   **Workflow:**

   **Get Available Transitions:**
   ```bash
   # Use mcp__plugin_atlassian_atlassian__getTransitionsForJiraIssue
   # Parameters:
   #   cloudId: from Step 0
   #   issueIdOrKey: $TICKET_ID
   # Returns: List of available transitions with id and name
   ```

   **Find "In Review" Transition:**

   Search for transition matching "In Review" (case-insensitive):
   - Exact match: "In Review"
   - Common variations: "Review", "Code Review", "In Code Review", "PR Review"

   Algorithm:
   ```bash
   # Pseudocode
   review_transition_id=""

   for transition in transitions:
     if transition.name.lower() == "in review":
       review_transition_id = transition.id
       break
     elif "review" in transition.name.lower():
       review_transition_id = transition.id
       break

   if review_transition_id is empty:
     # Transition not found - handle below
   ```

   **Execute Transition:**

   If "In Review" transition found:
   ```bash
   # Use mcp__plugin_atlassian_atlassian__transitionJiraIssue
   # Parameters:
   #   cloudId: from Step 0
   #   issueIdOrKey: $TICKET_ID
   #   transitionId: review_transition_id
   #   comment: "PR created: {PR_URL}\n\nAutomatically transitioned to In Review via creating-pr skill."
   ```

   **Success Output:**
   ```
   ✓ Jira ticket {TICKET_ID} transitioned to "In Review"
   ✓ Comment added with PR link
   ```

   **Error Handling:**

   - **Transition Not Found:**
     ```
     ✓ PR created successfully: {PR_URL}

     ⚠️ Could not find "In Review" transition for ticket {TICKET_ID}

     Available transitions from "{CURRENT_STATUS}":
     1. {transition_1_name} (id: {id})
     2. {transition_2_name} (id: {id})

     Select transition [1/2/...] or skip [s]:
     ```
     - If user selects number → Execute that transition
     - If user selects 's' → Skip status update, provide manual instructions

   - **Transition Failed:**
     ```
     ✓ PR created successfully: {PR_URL}

     ⚠️ Failed to update Jira ticket status

     Error: {error_message}

     Manual steps:
     1. Open ticket: https://{site}.atlassian.net/browse/{TICKET_ID}
     2. Transition to "In Review" manually
     3. Add PR link in comments: {PR_URL}
     ```

   - **Permission Denied:**
     ```
     ✓ PR created successfully: {PR_URL}

     ⚠️ You don't have permission to transition ticket {TICKET_ID}

     Please ask your Jira admin to transition the ticket to "In Review"
     PR link: {PR_URL}
     ```

   **Critical Principle:** Jira errors in Step 5 are NON-BLOCKING - PR was already created successfully

## Update Comment Templates

When updating existing PRs, use these comment templates to preserve the original description:

### General PR Update Template

```markdown
## 🔄 PR Update

**Commit**: `<commit-sha>` - `<commit-message>`

### Changes Made

- [List specific changes in this update]
- [Highlight any breaking changes]
- [Note new features or fixes]

### Impact

- [Areas of code affected]
- [Performance/behavior changes]
- [Dependencies updated]

### Testing

- [How to test these changes]
- [Regression testing notes]

### Next Steps

- [Remaining work if any]
- [Items for review focus]

```

### Critical Fix Update Template

```markdown
## 🚨 Critical Fix Applied

**Commit**: `<commit-sha>` - `<commit-message>`

### Issue Addressed

[Description of critical issue fixed]

### Solution

[Technical approach taken]

### Verification Steps

1. [Step to reproduce original issue]
2. [Step to verify fix]
3. [Regression test steps]

### Risk Assessment

- **Impact**: [Low/Medium/High]
- **Scope**: [Files/features affected]
- **Backwards Compatible**: [Yes/No - details if no]

```

### Feature Enhancement Template

```markdown
## ✨ Feature Enhancement

**Commit**: `<commit-sha>` - `<commit-message>`

### Enhancement Details

[Description of feature improvement/addition]

### Technical Implementation

- [Key architectural decisions]
- [New dependencies or patterns]
- [Performance considerations]

### User Experience Impact

[How this affects end users]

### Testing Strategy

[Approach to testing this enhancement]

```

## Action Points Extraction

When converting Jira ticket descriptions to PR checklists, identify action items in various formats.

### Common Action Point Patterns

**1. Numbered Lists:**
```
1. Implement user login
2. Add password validation
3. Create session management
```

**2. Bullet Points:**
```
- Set up authentication endpoints
- Configure JWT tokens
- Add rate limiting
```

**3. Checkbox Format:**
```
[ ] Database migrations
[ ] API endpoints
[ ] Unit tests
```

**4. Verb-Based Items:**

Look for lines starting with action verbs:
- "Implement...", "Add...", "Create...", "Update...", "Fix...", "Remove..."
- "Ensure...", "Verify...", "Test...", "Configure...", "Setup..."

**5. Acceptance Criteria Section:**
```
Acceptance Criteria:
- Users can login with email/password
- Invalid credentials show error message
- Sessions expire after 24 hours
```

### Conversion Algorithm

1. **Split content** into summary (non-actionable) and action items
2. **Identify action patterns** using formats above
3. **Convert to checklist** format:
   ```markdown
   - [ ] Action item 1
   - [ ] Action item 2
   ```
4. **Preserve non-action content** in Summary section
5. **Move action items** to Test Plan section as checkboxes

### Example Conversion

**Jira Description Input:**
```
Implement user authentication system.

Requirements:
1. Email/password login
2. JWT token generation
3. Session management

Technical Details:
- Use bcrypt for password hashing
- Tokens expire after 24h
- Store sessions in Redis

Acceptance Criteria:
- Users can login successfully
- Invalid credentials rejected
- Sessions persist across requests
```

**Converted PR Body Output:**
```markdown
## Summary

Implement user authentication system.

**Technical Details:**
- Use bcrypt for password hashing
- Tokens expire after 24h
- Store sessions in Redis

## Jira Ticket

**Ticket:** [PROJ-123](https://site.atlassian.net/browse/PROJ-123)
**Current Status:** To Do

## Test Plan

- [ ] Email/password login
- [ ] JWT token generation
- [ ] Session management
- [ ] Users can login successfully
- [ ] Invalid credentials rejected
- [ ] Sessions persist across requests

```

## Error Handling Summary

### Critical Principle

**Jira integration failures should NEVER block PR creation.**

**Execution Order:**
1. Always create the PR (core functionality)
2. Then attempt Jira status update (enhancement)
3. If Jira fails → Warn user, provide manual steps

### Error Scenarios

All error scenarios are documented in Step 0 and Step 5 of the workflow.

**Step 0 Errors (Ticket Fetch):**
- Ticket not found (404) → Ask to continue without Jira
- Permission denied (403) → Ask to continue without Jira
- Network/API errors → Ask to continue without Jira

**Step 5 Errors (Status Update):**
- Transition not found → List options, ask user to select
- Transition failed → Provide manual instructions
- Permission denied → Inform user, provide ticket URL

**Fallback Behavior:**
```
try:
  fetch_jira_ticket()
  jira_data_available = true
except:
  warn_user()
  ask_continue_without_jira()
  jira_data_available = false

# Always execute PR creation
create_pr(use_jira_data=jira_data_available)

# Only update Jira if available
if jira_data_available and pr_created_successfully:
  try:
    update_jira_status()
  except:
    warn_user()
    provide_manual_instructions()
```

## Status Transition Logic

Different Jira workflows have different paths to "In Review". This skill handles various workflow configurations.

### Transition Discovery Process

**Step 1: Fetch Available Transitions**
```bash
# Use mcp__plugin_atlassian_atlassian__getTransitionsForJiraIssue
# Returns transitions available from CURRENT status
```

Example response:
```json
{
  "transitions": [
    {"id": "31", "name": "Start Progress"},
    {"id": "41", "name": "In Review"},
    {"id": "51", "name": "Done"}
  ]
}
```

**Step 2: Match "In Review" Transition**

Search algorithm (in order of preference):
1. Exact match (case-insensitive): "In Review"
2. Contains "review": "Code Review", "PR Review", "Peer Review"
3. Shortened: "Review"
4. Alternative workflows: "Ready for Review", "Under Review"

**Matching Logic:**
```python
def find_review_transition(transitions):
    # Exact match first
    for t in transitions:
        if t.name.lower() == "in review":
            return t.id

    # Contains "review"
    for t in transitions:
        if "review" in t.name.lower():
            return t.id

    # Not found
    return None
```

**Step 3: Execute Transition**

If match found:
```bash
# Use mcp__plugin_atlassian_atlassian__transitionJiraIssue
# Parameters:
#   cloudId, issueIdOrKey, transitionId
#   comment: "PR created: {PR_URL}\n\nTransitioned via creating-pr skill."
```

### Common Workflow Patterns

**Pattern 1: Simple Kanban**
```
To Do → In Progress → In Review → Done
```
- Transitions from "To Do": ["In Progress"]
- Transitions from "In Progress": ["In Review", "To Do"]
- **Solution:** Only transition when in "In Progress" status

**Pattern 2: Extended Workflow**
```
Backlog → To Do → In Progress → Code Review → QA → Done
```
- "Code Review" = "In Review" equivalent
- **Solution:** Match "Code Review" using contains("review") logic

**Pattern 3: Simplified Workflow**
```
To Do → In Progress → Done
```
- No "In Review" status exists
- **Solution:** Ask user to map to available transition or skip

### Multi-Step Transition Strategy

**Conservative Approach (Recommended):**
- Only transition if "In Review" is directly available from current status
- Do NOT perform multi-step transitions automatically
- If not available, ask user or skip

**Example:**
```
Current Status: "To Do"
Available Transitions: ["In Progress"]
"In Review" not available from "To Do"

Action: Don't transition automatically
        Ask user: "Select transition [1] In Progress or [s] skip"
```

### Transition with Comment

Always include PR link in transition comment:
```markdown
PR created: {PR_URL}

This ticket has been transitioned to "In Review" automatically by the creating-pr skill.

**Next Steps:**
- Review the PR for code quality
- Run CI/CD checks
- Request changes if needed
- Approve and merge when ready

```

This provides context in Jira audit trail and notifies watchers.

## Example Usage Patterns

### Creating PR:

1. Create branch and make changes
2. Stage, commit, push → triggers PR creation
3. Each subsequent push triggers update comment

### Commit Message Conventions

- `feat:` - New features
- `fix:` - Bug fixes
- `refactor:` - Code refactoring
- `docs:` - Documentation changes
- `test:` - Test additions/modifications
- `chore:` - Maintenance tasks
- `style:` - Formatting changes

### Branch Naming Conventions

- `feature/description` - New features
- `fix/bug-description` - Bug fixes
- `refactor/component-name` - Code refactoring
- `docs/update-readme` - Documentation updates
- `test/add-unit-tests` - Test additions