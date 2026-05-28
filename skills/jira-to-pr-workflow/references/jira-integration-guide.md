# Jira Integration Guide

Detailed patterns and algorithms for Jira integration in jira-to-pr-workflow.

## Atlassian MCP Tools Usage

### Tool: getAccessibleAtlassianResources

**Purpose:** Get cloudId for subsequent Jira API calls

**Usage:**
```
Tool: mcp__plugin_atlassian_atlassian__getAccessibleAtlassianResources
Parameters: None
Returns: Array of accessible Atlassian resources
```

**Response Example:**
```json
[
  {
    "id": "abc123-def456-ghi789",
    "name": "Company Jira",
    "url": "https://company.atlassian.net",
    "scopes": ["read:jira-user", "read:jira-work", "write:jira-work"],
    "avatarUrl": "https://..."
  }
]
```

**Extract cloudId:**
```
cloudId = response[0].id  # Usually first resource
site_url = response[0].url
```

### Tool: getJiraIssue

**Purpose:** Fetch ticket details by ID or key

**Usage:**
```
Tool: mcp__plugin_atlassian_atlassian__getJiraIssue
Parameters:
  cloudId: "abc123-def456-ghi789"  # from getAccessibleAtlassianResources
  issueIdOrKey: "IOS-2345"
  fields: ["summary", "description", "status", "issuetype", "priority", "key", "assignee", "labels", "components"]
Returns: Jira issue object
```

**Response Example:**
```json
{
  "id": "10001",
  "key": "IOS-2345",
  "fields": {
    "summary": "Implement user profile screen",
    "description": "Create a new profile screen with user details...\n\nAcceptance Criteria:\n1. Display user name and email\n2. Show profile photo\n3. Allow editing of bio",
    "status": {
      "name": "To Do",
      "id": "10000",
      "statusCategory": {
        "key": "new"
      }
    },
    "issuetype": {
      "name": "Story",
      "id": "10001",
      "subtask": false
    },
    "priority": {
      "name": "High",
      "id": "2"
    },
    "assignee": {
      "displayName": "John Doe",
      "accountId": "xyz789"
    },
    "labels": ["mobile", "ui", "swift"],
    "components": [
      {"name": "iOS App"}
    ]
  }
}
```

**Field Extraction:**
```
ticket_id = response.key
summary = response.fields.summary
description = response.fields.description
status = response.fields.status.name
issue_type = response.fields.issuetype.name
priority = response.fields.priority.name
labels = response.fields.labels
components = [c.name for c in response.fields.components]
```

### Tool: getTransitionsForJiraIssue

**Purpose:** Get available status transitions for ticket

**Usage:**
```
Tool: mcp__plugin_atlassian_atlassian__getTransitionsForJiraIssue
Parameters:
  cloudId: "abc123-def456-ghi789"
  issueIdOrKey: "IOS-2345"
Returns: Array of available transitions
```

**Response Example:**
```json
{
  "transitions": [
    {
      "id": "11",
      "name": "Start Progress",
      "to": {
        "name": "In Progress",
        "id": "3"
      }
    },
    {
      "id": "21",
      "name": "In Review",
      "to": {
        "name": "In Review",
        "id": "10001"
      }
    }
  ]
}
```

### Tool: transitionJiraIssue

**Purpose:** Change ticket status

**Usage:**
```
Tool: mcp__plugin_atlassian_atlassian__transitionJiraIssue
Parameters:
  cloudId: "abc123-def456-ghi789"
  issueIdOrKey: "IOS-2345"
  transition:
    id: "21"  # from getTransitionsForJiraIssue
  fields: {}  # optional field updates
Returns: Success or error
```

### Tool: addCommentToJiraIssue

**Purpose:** Add comment to ticket

**Usage:**
```
Tool: mcp__plugin_atlassian_atlassian__addCommentToJiraIssue
Parameters:
  cloudId: "abc123-def456-ghi789"
  issueIdOrKey: "IOS-2345"
  commentBody: "PR created: https://github.com/..."
Returns: Comment object
```

## Ticket Data Extraction Patterns

### Summary Extraction

**Purpose:** Use as feature name and PR title

**Pattern:**
```
summary = ticket.fields.summary
feature_name = summary  # Use as-is

# Example:
# "Implement user profile screen" → Feature: Implement user profile screen
```

### Description Parsing

**Purpose:** Extract requirements and acceptance criteria for PRD

**Parsing Algorithm:**

```python
def parse_jira_description(description):
    """
    Parse Jira description into structured data.

    Returns:
      - summary_content: Non-actionable text
      - requirements: Action items and requirements
      - acceptance_criteria: Specific acceptance criteria
    """

    lines = description.split('\n')

    summary_content = []
    requirements = []
    acceptance_criteria = []

    current_section = 'summary'

    for line in lines:
        # Detect section headers
        if re.match(r'(?i)requirements?:', line):
            current_section = 'requirements'
            continue
        elif re.match(r'(?i)acceptance criteria:', line):
            current_section = 'acceptance'
            continue
        elif re.match(r'(?i)technical details?:', line):
            current_section = 'summary'
            continue

        # Detect action items
        if re.match(r'^\d+\.', line):  # Numbered list
            if current_section == 'acceptance':
                acceptance_criteria.append(line)
            else:
                requirements.append(line)
        elif re.match(r'^[-*]', line):  # Bullet list
            if current_section == 'acceptance':
                acceptance_criteria.append(line)
            else:
                requirements.append(line)
        elif re.match(r'^\[\s*\]', line):  # Checkbox
            if current_section == 'acceptance':
                acceptance_criteria.append(line)
            else:
                requirements.append(line)
        elif re.match(r'^(Implement|Add|Create|Update|Fix|Remove|Ensure|Verify)', line):
            # Verb-based action items
            if current_section == 'acceptance':
                acceptance_criteria.append(line)
            else:
                requirements.append(line)
        else:
            # Non-action content
            summary_content.append(line)

    return {
        'summary': '\n'.join(summary_content).strip(),
        'requirements': requirements,
        'acceptance_criteria': acceptance_criteria
    }
```

**Example:**

**Input (Jira Description):**
```
Implement user profile screen with editable fields.

Technical Details:
- Use SwiftUI for UI
- Store data in CoreData
- Support offline mode

Requirements:
1. Display user name and email
2. Show profile photo with upload
3. Allow editing of bio field

Acceptance Criteria:
- Users can view their profile
- Users can edit bio and save
- Changes persist after app restart
- Offline edits sync when online
```

**Parsed Output:**
```python
{
  'summary': 'Implement user profile screen with editable fields.\n\nTechnical Details:\n- Use SwiftUI for UI\n- Store data in CoreData\n- Support offline mode',

  'requirements': [
    '1. Display user name and email',
    '2. Show profile photo with upload',
    '3. Allow editing of bio field'
  ],

  'acceptance_criteria': [
    '- Users can view their profile',
    '- Users can edit bio and save',
    '- Changes persist after app restart',
    '- Offline edits sync when online'
  ]
}
```

### Labels and Components Extraction

**Purpose:** Additional context for complexity analysis

**Pattern:**
```
labels = ticket.fields.labels  # Array of strings
components = [c.name for c in ticket.fields.components]  # Array of component names

# Use for complexity scoring:
# - labels like "complex", "migration", "breaking-change" → +3 to score
# - components count → Estimate # of areas affected
```

## Complexity Analysis Algorithm

### Scoring Formula

```python
def calculate_complexity_score(ticket):
    """
    Calculate complexity score from Jira ticket data.

    Score Breakdown:
    - Issue Type: Epic=10, Story=5, Task=2, Bug=1
    - Priority: High/Critical=3, Medium=1, Low=0
    - Components/Files: +2 per mention
    - Integrations: +3 per mention
    - Data Model Changes: +5 if detected
    - API Changes: +4 if detected
    - Complex Logic: +3 if detected
    - Acceptance Criteria: +1 per item
    - Labels: +3 for complexity indicators

    Returns: Integer score
    """

    score = 0

    # Issue Type
    issue_type = ticket.fields.issuetype.name
    if issue_type == "Epic":
        score += 10
    elif issue_type == "Story":
        score += 5
    elif issue_type == "Task":
        score += 2
    elif issue_type == "Bug":
        score += 1

    # Priority
    priority = ticket.fields.priority.name
    if priority in ["High", "Critical"]:
        score += 3
    elif priority == "Medium":
        score += 1

    # Parse description for indicators
    description = ticket.fields.description.lower()

    # Count components/files mentioned
    file_patterns = [r'\w+\.swift', r'\w+\.kt', r'\w+View', r'\w+Model', r'\w+Service']
    for pattern in file_patterns:
        matches = re.findall(pattern, description, re.IGNORECASE)
        score += len(matches) * 2

    # Detect integrations
    integration_keywords = ['api', 'endpoint', 'service', 'integration', 'webhook', 'graphql', 'rest']
    for keyword in integration_keywords:
        if keyword in description:
            score += 3
            break  # Count once

    # Detect data model changes
    data_keywords = ['database', 'schema', 'migration', 'model', 'coredata', 'realm', 'entity']
    if any(keyword in description for keyword in data_keywords):
        score += 5

    # Detect API changes
    api_keywords = ['api', 'endpoint', 'route', 'controller', 'graphql', 'mutation', 'query']
    if any(keyword in description for keyword in api_keywords):
        score += 4

    # Detect complex logic
    complexity_keywords = ['algorithm', 'calculation', 'complex', 'optimization', 'performance']
    if any(keyword in description for keyword in complexity_keywords):
        score += 3

    # Count acceptance criteria
    parsed = parse_jira_description(ticket.fields.description)
    score += len(parsed['acceptance_criteria'])

    # Check labels for complexity indicators
    complexity_labels = ['complex', 'migration', 'breaking-change', 'refactor']
    for label in ticket.fields.labels:
        if label.lower() in complexity_labels:
            score += 3

    return score
```

### Threshold Interpretation

```python
def interpret_complexity(score):
    """
    Interpret complexity score and provide recommendation.

    Thresholds:
    - 0-5: Simple
    - 6-12: Moderate
    - 13+: Complex
    """

    if score <= 5:
        return {
            'level': 'simple',
            'recommendation': 'minimal',
            'description': 'Simple task with clear scope and few dependencies'
        }
    elif score <= 12:
        return {
            'level': 'moderate',
            'recommendation': 'standard',
            'description': 'Moderate complexity with multiple components or some integration'
        }
    else:
        return {
            'level': 'complex',
            'recommendation': 'detailed',
            'description': 'Complex task requiring detailed breakdown and planning'
        }
```

### Analysis Report Format

```python
def generate_analysis_report(ticket, score, interpretation):
    """
    Generate user-facing complexity analysis report.
    """

    # Extract contributing factors
    parsed = parse_jira_description(ticket.fields.description)

    # Count components mentioned
    description = ticket.fields.description
    components = re.findall(r'\w+(?:View|Model|Service|Manager|Controller)', description, re.IGNORECASE)
    files = re.findall(r'\w+\.(?:swift|kt|java|ts)', description, re.IGNORECASE)

    report = f"""
Based on ticket {ticket.key} analysis:
  - Complexity Score: {score}
  - Issue Type: {ticket.fields.issuetype.name}
  - Priority: {ticket.fields.priority.name}
  - Estimated Components: {', '.join(set(components)) if components else 'Not specified'}
  - Estimated Files: {', '.join(set(files)) if files else 'Not specified'}
  - Acceptance Criteria: {len(parsed['acceptance_criteria'])} items
  - Recommended Approach: {interpretation['recommendation'].capitalize()} breakdown

Complexity Assessment: {interpretation['description']}

Contributing Factors:
"""

    # Add scoring breakdown
    if ticket.fields.issuetype.name == "Epic":
        report += "  - Epic-level task (+10)\n"
    elif ticket.fields.issuetype.name == "Story":
        report += "  - Story-level task (+5)\n"

    if ticket.fields.priority.name in ["High", "Critical"]:
        report += f"  - {ticket.fields.priority.name} priority (+3)\n"

    if components:
        report += f"  - {len(set(components))} components identified (+{len(set(components)) * 2})\n"

    if 'api' in description.lower() or 'endpoint' in description.lower():
        report += "  - API changes detected (+4)\n"

    if any(kw in description.lower() for kw in ['database', 'schema', 'migration']):
        report += "  - Data model changes (+5)\n"

    if len(parsed['acceptance_criteria']) > 0:
        report += f"  - {len(parsed['acceptance_criteria'])} acceptance criteria (+{len(parsed['acceptance_criteria'])})\n"

    return report
```

## Error Recovery Strategies

### Ticket Not Found (404)

**Detection:**
```
HTTP Status: 404
Error: Issue does not exist
```

**Recovery:**
```
1. Validate ticket ID format: [A-Z]+-[0-9]+
2. If valid format:
   - Possible ticket deleted
   - Possible wrong project
   - Possible access restriction
3. Ask user: "Continue without Jira? [y/n]"
4. If yes: Set JIRA_ENABLED=false, continue to complexity analysis
5. If no: Exit skill
```

### Permission Denied (403)

**Detection:**
```
HTTP Status: 403
Error: You do not have permission to view this issue
```

**Recovery:**
```
1. Check user has read access to project
2. Ask user: "Continue without Jira? [y/n]"
3. If yes: Set JIRA_ENABLED=false, prompt for manual feature description
4. If no: Exit skill
```

### Network/API Errors

**Detection:**
```
Connection timeout
Network unreachable
API rate limit exceeded
```

**Recovery:**
```
1. Log error details
2. Retry once after 2 seconds
3. If still fails:
   - Display error message
   - Ask user: "Continue without Jira? [y/n]"
4. If yes: Set JIRA_ENABLED=false, continue
5. If no: Exit skill
```

### Invalid Ticket ID Format

**Detection:**
```
Regex match fails: ^[A-Z]+-[0-9]+$
```

**Recovery:**
```
1. Display format error:
   "⚠️  Invalid ticket ID format: {input}
    Expected format: PROJECT-123 (e.g., IOS-456, MOBILE-789)"
2. Ask user to re-enter or skip:
   "Enter valid ticket ID or press Enter to skip:"
3. If re-entered: Validate again
4. If skipped: Set JIRA_ENABLED=false, continue
```

## State Persistence

### jira-context.json Schema

**Location:** `.claude/feature-state/jira-{TICKET_KEY}/jira-context.json`

**Purpose:** Persist Jira data for checkpoint resume

**Schema:**
```json
{
  "ticket_id": "IOS-2345",
  "ticket_url": "https://company.atlassian.net/browse/IOS-2345",
  "cloudId": "abc123-def456-ghi789",
  "site_url": "https://company.atlassian.net",
  "summary": "Implement user profile screen",
  "description": "Create a new profile screen...",
  "parsed_description": {
    "summary": "Create a new profile screen...",
    "requirements": [
      "1. Display user name and email",
      "2. Show profile photo with upload"
    ],
    "acceptance_criteria": [
      "- Users can view their profile",
      "- Users can edit bio and save"
    ]
  },
  "issue_type": "Story",
  "priority": "High",
  "status": "To Do",
  "labels": ["mobile", "ui", "swift"],
  "components": ["iOS App"],
  "complexity_score": 14,
  "complexity_level": "complex",
  "jira_enabled": true,
  "fetched_at": "2026-01-27T10:30:00Z"
}
```

### Save Context

```python
def save_jira_context(ticket, score, interpretation, feature_slug):
    """
    Save Jira context for checkpoint resume.
    """

    parsed = parse_jira_description(ticket.fields.description)

    context = {
        'ticket_id': ticket.key,
        'ticket_url': f"{site_url}/browse/{ticket.key}",
        'cloudId': cloudId,
        'site_url': site_url,
        'summary': ticket.fields.summary,
        'description': ticket.fields.description,
        'parsed_description': parsed,
        'issue_type': ticket.fields.issuetype.name,
        'priority': ticket.fields.priority.name,
        'status': ticket.fields.status.name,
        'labels': ticket.fields.labels,
        'components': [c.name for c in ticket.fields.components],
        'complexity_score': score,
        'complexity_level': interpretation['level'],
        'jira_enabled': True,
        'fetched_at': datetime.utcnow().isoformat() + 'Z'
    }

    path = f".claude/feature-state/{feature_slug}/jira-context.json"
    with open(path, 'w') as f:
        json.dump(context, f, indent=2)
```

### Load Context on Resume

```python
def load_jira_context(feature_slug):
    """
    Load Jira context from checkpoint.
    """

    path = f".claude/feature-state/{feature_slug}/jira-context.json"

    if not os.path.exists(path):
        return None

    with open(path, 'r') as f:
        context = json.load(f)

    return context
```

## Transition Matching Patterns

### Common Workflow Variations

**Pattern 1: Simple Kanban**
```
To Do → In Progress → In Review → Done
```
- Direct "In Review" transition
- Match: "in review" (exact, case-insensitive)

**Pattern 2: Extended Workflow**
```
Backlog → To Do → In Progress → Code Review → QA → Done
```
- "Code Review" = "In Review" equivalent
- Match: "code review" (contains "review")

**Pattern 3: GitHub-style**
```
To Do → In Progress → Ready for Review → Done
```
- "Ready for Review" variant
- Match: "ready for review" (contains "review")

**Pattern 4: Peer Review**
```
To Do → Development → Peer Review → Testing → Done
```
- "Peer Review" variant
- Match: "peer review" (contains "review")

**Pattern 5: Simplified**
```
To Do → In Progress → Done
```
- No review status
- Match: None → Ask user to select or skip

### Matching Algorithm

```python
def find_review_transition(transitions):
    """
    Find "In Review" transition or closest variant.

    Priority:
    1. Exact match: "In Review"
    2. Contains "review": "Code Review", "Peer Review", etc.
    3. Shortened: "Review"
    4. None found
    """

    # Priority 1: Exact match
    for t in transitions:
        if t['name'].lower() == 'in review':
            return t['id']

    # Priority 2: Contains "review"
    review_transitions = []
    for t in transitions:
        if 'review' in t['name'].lower():
            review_transitions.append(t)

    if len(review_transitions) == 1:
        return review_transitions[0]['id']
    elif len(review_transitions) > 1:
        # Multiple review transitions, prefer "Code Review"
        for t in review_transitions:
            if 'code' in t['name'].lower():
                return t['id']
        # Otherwise return first
        return review_transitions[0]['id']

    # Priority 3: None found
    return None
```

### User Selection on Ambiguity

```python
def handle_transition_not_found(transitions, ticket_id, current_status):
    """
    Handle case when "In Review" transition not found.
    """

    print(f"""
✓ PR created successfully: {PR_URL}

⚠️  Could not find "In Review" transition for ticket {ticket_id}

Current status: {current_status}

Available transitions:
""")

    for i, t in enumerate(transitions, 1):
        print(f"{i}. {t['name']} → {t['to']['name']}")

    print(f"{len(transitions) + 1}. Skip (manual transition later)")

    choice = input("\nSelect transition [1-{len(transitions) + 1}]: ")

    if choice.isdigit() and 1 <= int(choice) <= len(transitions):
        selected = transitions[int(choice) - 1]
        return selected['id']
    else:
        # Skip
        print(f"""
Manual steps:
1. Open ticket: {site_url}/browse/{ticket_id}
2. Transition to appropriate review status
3. Add PR link in comments: {PR_URL}
""")
        return None
```
