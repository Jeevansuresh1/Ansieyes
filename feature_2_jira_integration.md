# Feature 2: JIRA Integration

## Problem
After issue triage, teams must manually create JIRA tickets. This adds friction and the triage insights (type, severity, analysis) are lost in translation.

## Solution
Auto-create JIRA tickets after successful triage. Maps Ansieyes type/severity to JIRA issue type/priority.

## Architecture

```
\ansieyes_triage
    |
    v
issue_triager.triage_issue()
    |
    +-- prompt injection check
    +-- duplicate check (if duplicate -> skip JIRA)
    +-- librarian + surgeon analysis
    |
    v
Labels applied to GitHub issue
    |
    v
[NEW] jira_connector.create_ticket()
    |
    +-- Maps type: Bug -> Bug, Enhancement -> Story, Feature -> Story, Other -> Task
    +-- Maps severity: Critical/High/Medium/Low -> JIRA priority
    +-- Creates ticket with title, description, link back to GitHub issue
    +-- Posts JIRA ticket URL as GitHub comment
    +-- Adds JIRA key as label (e.g., PROJ-123)
```

## New File: `jira_connector.py`

```python
class JiraConnector:
    __init__(self)           # Reads env vars, init client if enabled
    is_enabled() -> bool     # Check JIRA_ENABLED
    create_ticket(triage_result, issue_url, issue_number, repo_name) -> dict
    _map_type_to_jira(type) -> str
    _map_severity_to_jira(severity) -> str
    _build_description(triage_result, issue_url) -> str
```

## Environment Variables
```
JIRA_ENABLED=false          # Must be explicitly enabled
JIRA_URL=https://yourname.atlassian.net
JIRA_EMAIL=your@email.com
JIRA_API_TOKEN=your_token   # Generate at id.atlassian.com/manage-profile/security/api-tokens
JIRA_PROJECT_KEY=PROJ        # Project key in JIRA (e.g., ANS, PROJ)
```

## Type/Priority Mapping

| Ansieyes Type | JIRA Issue Type |
|--------------|-----------------|
| Bug | Bug |
| Enhancement | Story |
| Feature Request | Story |
| Other/Unknown | Task |

| Ansieyes Severity | JIRA Priority |
|------------------|---------------|
| Critical | Highest |
| High | High |
| Medium | Medium |
| Low | Low |
| Unknown | Medium |

## JIRA Ticket Content

**Summary**: `[GitHub #{issue_number}] {issue_title}`
**Description**:
```
h2. GitHub Issue
{issue_title}

{issue_description (truncated to 2000 chars)}

h2. AI Triage Results
* *Type*: {type}
* *Severity*: {severity}
* *GitHub Link*: {issue_url}
* *Repository*: {repo_name}

h2. Analysis Summary
{surgeon_summary (first 500 chars)}

----
_Auto-created by Ansieyes_
```

## Safety
- JIRA integration NEVER blocks triage. All JIRA calls wrapped in try/except.
- If JIRA_ENABLED=true but credentials are missing/invalid, log warning and skip.
- JIRA ticket creation happens AFTER triage is fully complete and labels are applied.
- If JIRA create fails, triage result is still posted successfully.

## Dependency
- `jira>=3.5.0` added to requirements.txt
