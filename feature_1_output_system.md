# Feature 1: Structured Output Template System

## Problem
All output formatting is inline string concatenation scattered across `app.py`, `issue_triager.py`, and `pr_reviewer.py`. This causes:
- Inconsistent formatting between different outputs
- No severity classification or visual hierarchy
- Walls of text with no collapsible sections
- Branding substitutions scattered across files

## Solution
New `output_formatter.py` module with pure rendering functions. Ask Gemini for structured data, render through consistent templates.

## Output Design (Based on Industry Research)

### PR Review Output Structure
```markdown
# Ansieyes Report

> PR: **#42 - Add user authentication** | Reviewed: 2026-05-16 12:30 UTC

## Dashboard
| Metric | Score |
|--------|-------|
| Security | :red_circle: 3/10 |
| Code Quality | :yellow_circle: 6/10 |
| Performance | :green_circle: 9/10 |
| **Overall** | **6/10** |

**Verdict**: APPROVE WITH COMMENTS

## Summary
Brief 2-3 sentence summary of what the PR does...

## Findings (3 issues)

### :red_circle: Critical: Hardcoded API key in auth.py
**File**: `auth.py:15` | **Category**: Security

> API key is hardcoded directly in source code...

**Suggested Fix**:
```diff
- API_KEY = "sk-abc123..."
+ API_KEY = os.getenv("API_KEY")
```

<details><summary>:yellow_circle: Minor: Missing error handling in login.py (2 findings)</summary>

... collapsible content for lower-severity findings ...

</details>

## Files Changed
| File | Changes | Status |
|------|---------|--------|
| auth.py | +50 -0 | :red_circle: 1 critical issue |
| login.py | +20 -5 | :yellow_circle: 2 minor issues |
| tests/test_auth.py | +30 -0 | :green_circle: No issues |

---
<sub>Powered by Ansieyes | AI-Issue-Triage</sub>
```

### Triage Report Output Structure
```markdown
# Ansieyes Report

> Issue: **#15 - Login fails on mobile** | Triaged: 2026-05-16 12:30 UTC

## Classification
| Field | Value |
|-------|-------|
| Type | :bug: `BUG` |
| Severity | :red_circle: `HIGH` |
| Confidence | 85% |

## Summary
2-3 sentence summary of what the issue is...

## Root Cause Analysis
Detailed analysis from Surgeon pass...

<details><summary>Relevant Files (5 identified by Librarian)</summary>

1. `src/auth/login.py` - Main login handler
2. `src/middleware/session.py` - Session management
...

</details>

## Proposed Solutions
1. Fix the session token validation in `login.py:42`
2. Add mobile user-agent detection...

---
<sub>Powered by Ansieyes | AI-Issue-Triage</sub>
```

## Functions to Implement

### Core Renderers
- `format_pr_review(review_data, file_changes, metadata)` -> str
- `format_triage_report(triage_result)` -> str
- `format_security_alert(injection_data)` -> str
- `format_duplicate_report(duplicate_data)` -> str
- `format_walkthrough(walkthrough_data)` -> str (Feature 3)
- `format_workflow_analysis(analysis, workflow_name, conclusion, failed_jobs, url)` -> str
- `format_fix_suggestions(fix_result)` -> str (Feature 4)
- `format_jira_created(jira_key, jira_url)` -> str (Feature 2)

### Status/Error Messages
- `format_processing_message(command)` -> str
- `format_error_message(error_type, details, suggestion)` -> str
- `format_unauthorized_message(username)` -> str
- `format_invalid_context_message(command, expected, actual, alternative)` -> str

### Shared Components (Private)
- `_severity_badge(severity)` -> emoji string
- `_collapsible(title, content, open)` -> details/summary HTML
- `_findings_table(findings)` -> markdown table
- `_header(title)` -> standardized header with timestamp
- `_footer()` -> standardized footer

## Migration Plan
1. Create `output_formatter.py` with all functions
2. Replace inline strings in `app.py` (13 create_comment calls)
3. Make `issue_triager.format_triage_comment()` delegate to formatter
4. Make `pr_reviewer.format_review_summary()` delegate to formatter
5. Delete `format_workflow_comment()` from `app.py`
