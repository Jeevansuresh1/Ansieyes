# Ansieyes Feature Expansion - Complete Implementation Plan

> **Created**: 2026-05-16
> **Purpose**: Self-contained plan for implementing 4 new features. Read this file and execute phase by phase.
> **Pre-requisite**: Read `context.md` in this repo for full project understanding.

---

## Table of Contents
1. [Implementation Order](#implementation-order)
2. [Phase 0: Bug Fix](#phase-0-bug-fix)
3. [Phase 1: Structured Output Template System](#phase-1-structured-output-template-system)
4. [Phase 2a: JIRA Integration](#phase-2a-jira-integration)
5. [Phase 2b: PR Walkthrough + Sequence Diagrams](#phase-2b-pr-walkthrough--sequence-diagrams)
6. [Phase 3: Auto-Fix Agent](#phase-3-auto-fix-agent)
7. [Phase 4: Documentation & Config Updates](#phase-4-documentation--config-updates)
8. [New File Structure](#new-file-structure)
9. [Verification Plan](#verification-plan)
10. [Risks & Mitigations](#risks--mitigations)

---

## Implementation Order

```
Phase 0: Bug fix (app.py review_pr auto-trigger) ............. [DO FIRST - prerequisite]
    |
Phase 1: output_formatter.py ................................. [FOUNDATION - all features need this]
    |
    +---> Phase 2a: jira_connector.py ........................ [Independent]
    |
    +---> Phase 2b: walkthrough_generator.py ................. [Independent]
    |
Phase 3: auto_fixer.py ....................................... [Depends on Phase 1]
    |
Phase 4: Docs, env_example.txt, requirements.txt ............. [Final cleanup]
```

**Why this order**: Feature 1 (output_formatter) is the foundation — every other feature renders output through it. Features 2a and 2b are independent of each other. Feature 4 depends on Feature 1's severity parsing.

---

## Phase 0: Bug Fix

### What's Broken

In `app.py`, there are TWO paths that trigger PR review:

1. **Auto-trigger** (`review_pr()` at line 305) — fires when a PR is opened/synchronized/reopened
2. **Mention-trigger** (`handle_pr_review_mention()` at line 847) — fires when user comments `\ansieyes_prreview`

Both call `pr_reviewer.review_pr()` which returns a **STRING** (markdown text).

The **mention-trigger** correctly treats it as a string (line 926: `review_text.startswith("❌")`, line 936: `issue.create_comment(review_text)`).

The **auto-trigger** INCORRECTLY treats it as a dict:
- Line 351: `review_comments = pr_reviewer.review_pr(...)` — stores string in `review_comments`
- Line 363: `post_review_comments(pr, review_comments)` — passes string to function expecting dict
- Line 375: `pr_reviewer.format_review_summary(review_comments)` — works (passthrough), but returns string
- Line 382: `review_comments.get('file_comments', [])` — **CRASHES** — calling `.get()` on a string

### Fix

**File: `app.py`**

**Step 1**: Rewrite `review_pr()` function (lines 305-368). Replace lines 351-367:

```python
# BEFORE (broken):
review_comments = pr_reviewer.review_pr(
    title=title,
    body=body,
    file_changes=file_changes,
    repo_url=repo_url
)

if not review_comments:
    logger.info("No review comments generated")
    return

# Post review comments
post_review_comments(pr, review_comments)

# AFTER (fixed):
review_text = pr_reviewer.review_pr(
    title=title,
    body=body,
    file_changes=file_changes,
    repo_url=repo_url
)

if not review_text or review_text.startswith("❌"):
    logger.info("No review generated or error occurred")
    return

# Post review as PR comment
pr.create_issue_comment(review_text)
logger.info("Posted review comment")
```

**Step 2**: Delete the entire `post_review_comments()` function (lines 371-400). It is no longer called by anything.

```python
# DELETE THIS ENTIRE FUNCTION (lines 371-400):
def post_review_comments(pr, review_comments):
    """Post review comments to the PR"""
    # ... all of it ...
```

---

## Phase 1: Structured Output Template System

### Goal
Replace all inline string concatenation across `app.py`, `issue_triager.py`, and `pr_reviewer.py` with a centralized `output_formatter.py` module that produces clean, professional, consistent GitHub comments.

### Design Principles (from industry research)
- **Severity classification** on every finding: 🔴 Critical, 🟠 Major, 🟡 Minor, 🔵 Trivial, ⚪ Info
- **Collapsible sections** (`<details><summary>`) so comments aren't walls of text
- **Tables** for file-by-file findings
- **Consistent branding** — same header/footer/structure every time regardless of AI output
- **Actionable** — suggest fixes, not just flag problems
- **Timestamp** on every report

### New File: `output_formatter.py`

Create at `/Users/jeevans/Desktop/Ansieyes/Ansieyes/output_formatter.py`

Estimated ~400-500 lines. All pure functions — take data in, return markdown string out. No side effects, no GitHub API calls.

#### Private Helper Functions

```python
from datetime import datetime, timezone


def _timestamp() -> str:
    """Returns current UTC timestamp string."""
    return datetime.now(timezone.utc).strftime('%Y-%m-%d %H:%M:%S UTC')


def _header(title: str, subtitle: str = "") -> str:
    """
    Standard report header.
    
    Output:
    # 🤖 Ansieyes Report
    > **Title** | Generated: 2026-05-16 12:30:00 UTC
    """
    header = f"# 🤖 {title}\n\n"
    if subtitle:
        header += f"> {subtitle} | Generated: `{_timestamp()}`\n\n"
    else:
        header += f"> Generated: `{_timestamp()}`\n\n"
    return header


def _footer() -> str:
    """
    Standard report footer.
    
    Output:
    ---
    <sub>🤖 Powered by Ansieyes | AI-Issue-Triage</sub>
    """
    return "\n\n---\n<sub>🤖 Powered by Ansieyes | AI-Issue-Triage</sub>"


def _severity_badge(severity: str) -> str:
    """
    Returns emoji-colored severity badge.
    
    Input: "critical", "high", "medium", "low", "info"
    Output: "🔴 `CRITICAL`", "🟠 `HIGH`", etc.
    """
    severity_map = {
        'critical': '🔴 `CRITICAL`',
        'high': '🟠 `HIGH`',
        'medium': '🟡 `MEDIUM`',
        'low': '🟢 `LOW`',
        'info': '⚪ `INFO`',
        'safe': '✅ `SAFE`',
    }
    return severity_map.get(severity.lower(), f'⚪ `{severity.upper()}`')


def _collapsible(title: str, content: str, open_by_default: bool = False) -> str:
    """
    Wraps content in a collapsible <details> section.
    
    Output:
    <details open>
    <summary>Title</summary>
    
    Content here...
    
    </details>
    """
    open_attr = " open" if open_by_default else ""
    return f"<details{open_attr}>\n<summary>{title}</summary>\n\n{content}\n\n</details>\n"


def _findings_table(findings: list) -> str:
    """
    Renders a markdown table of findings.
    
    Input: [{"file": "auth.py", "severity": "critical", "description": "Hardcoded key"}]
    
    Output:
    | File | Severity | Finding |
    |------|----------|---------|
    | `auth.py` | 🔴 `CRITICAL` | Hardcoded key |
    """
    if not findings:
        return ""
    table = "| File | Severity | Finding |\n|------|----------|--------|\n"
    for f in findings:
        sev = _severity_badge(f.get('severity', 'info'))
        table += f"| `{f.get('file', 'unknown')}` | {sev} | {f.get('description', '')} |\n"
    return table


def _metric_field(emoji: str, label: str, value: str) -> str:
    """
    Returns a single metric line.
    
    Output: 🔴 **Risk Level:** `CRITICAL`
    """
    return f"{emoji} **{label}:** `{value}`  \n"
```

#### Core Renderer: PR Review

```python
def format_pr_review(review_text: str, file_changes: list = None, metadata: dict = None) -> str:
    """
    Wraps raw AI review text in the standardized Ansieyes template.
    
    Args:
        review_text: Raw markdown from AI-Issue-Triage cli.pr_review
        file_changes: List of file change dicts (filename, status, additions, deletions)
        metadata: Optional dict with pr_number, pr_title, repo_name
    
    Returns:
        Formatted markdown string
    
    The AI review text is used as-is inside the template (passthrough mode).
    We wrap it with: header -> file changes table -> AI review (in collapsible) -> footer
    
    Post-processing: Replace "AI Code Review" / "Gemini" branding with "Ansieyes"
    """
    metadata = metadata or {}
    pr_title = metadata.get('pr_title', '')
    pr_number = metadata.get('pr_number', '')
    
    subtitle = ""
    if pr_number and pr_title:
        subtitle = f"**PR #{pr_number}** — {pr_title}"
    
    comment = _header("Ansieyes PR Review", subtitle)
    
    # File changes summary table
    if file_changes:
        comment += "## Files Changed\n\n"
        comment += "| File | Status | Changes |\n|------|--------|--------|\n"
        for fc in file_changes:
            fname = fc.get('filename', 'unknown')
            status = fc.get('status', 'modified')
            adds = fc.get('additions', 0)
            dels = fc.get('deletions', 0)
            status_emoji = {'added': '🟢', 'removed': '🔴', 'modified': '🟡', 'renamed': '🔵'}.get(status, '⚪')
            comment += f"| `{fname}` | {status_emoji} {status} | +{adds} -{dels} |\n"
        comment += "\n"
    
    # AI Review content
    comment += "## Review\n\n"
    
    # Clean up AI branding
    cleaned = review_text
    cleaned = cleaned.replace("## 🤖 AI Code Review (Powered by Gemini)", "")
    cleaned = cleaned.replace("# 🤖 Gemini Analysis Report", "")
    cleaned = cleaned.replace("## 🤖 Ansieyes Report", "")
    cleaned = cleaned.replace("# 🤖 Ansieyes Report", "")
    cleaned = cleaned.strip()
    
    comment += cleaned
    comment += _footer()
    
    return comment
```

#### Core Renderer: Triage Report

```python
def format_triage_report(triage_result: dict) -> str:
    """
    Formats complete triage results as a GitHub comment.
    Routes to appropriate sub-formatter based on result type.
    
    Args:
        triage_result: Dict from IssueTriager.triage_issue() with keys:
            - prompt_injection_check
            - duplicate_check
            - librarian
            - surgeon
    
    Returns:
        Formatted markdown string
    """
    if not triage_result:
        return format_error_message("analysis", "No triage results available.")
    
    # Case 1: Prompt injection blocked (HIGH/CRITICAL)
    if triage_result.get("prompt_injection_check"):
        injection = triage_result["prompt_injection_check"]
        risk_level = injection.get("risk_level", "").lower()
        if injection.get("is_injection") and risk_level in ['high', 'critical']:
            return format_security_alert(injection)
    
    # Case 2: Duplicate detected
    if triage_result.get("duplicate_check"):
        dup = triage_result["duplicate_check"]
        if dup.get("is_duplicate"):
            return format_duplicate_report(dup)
    
    # Case 3: Normal triage with surgeon results
    if not triage_result.get("surgeon"):
        return format_error_message("analysis", "No analysis results available.")
    
    surg = triage_result["surgeon"]
    
    if "error" in surg:
        return format_error_message("analysis", surg['error'])
    
    # Use surgeon's formatted_output if available
    if "formatted_output" in surg:
        return _format_surgeon_report(surg, triage_result)
    
    return format_error_message("analysis", "No formatted output available.")


def _format_surgeon_report(surgeon_data: dict, triage_result: dict) -> str:
    """
    Formats the main surgeon analysis report.
    
    Takes the raw formatted_output from surgeon and wraps it in the Ansieyes template.
    Extracts type/severity using regex for the classification header.
    """
    import re
    
    formatted = surgeon_data.get("formatted_output", "")
    
    # Extract type and severity from formatted text
    type_match = re.search(r'\*\*Type:\*\*\s+`([^`]+)`', formatted)
    issue_type = type_match.group(1) if type_match else "Unknown"
    
    severity_match = re.search(r'\*\*Severity:\*\*\s+`([^`]+)`', formatted)
    severity = severity_match.group(1) if severity_match else "Unknown"
    
    comment = _header("Ansieyes Triage Report")
    
    # Classification box
    comment += "## Classification\n\n"
    comment += "| Field | Value |\n|-------|-------|\n"
    
    type_emoji = {'bug': '🐛', 'enhancement': '✨', 'feature_request': '🚀', 'feature request': '🚀'}.get(issue_type.lower(), '📋')
    comment += f"| Type | {type_emoji} `{issue_type.upper()}` |\n"
    comment += f"| Severity | {_severity_badge(severity)} |\n"
    comment += "\n"
    
    # Prompt injection info (if low/medium detected but not blocked)
    if triage_result.get("prompt_injection_check"):
        inj = triage_result["prompt_injection_check"]
        if inj.get("is_injection") and inj.get("risk_level", "").lower() in ['low', 'medium']:
            comment += f"> ℹ️ Low-risk prompt injection patterns detected (risk: {inj['risk_level']}) — analysis continued.\n\n"
    
    # Librarian results (collapsible)
    if triage_result.get("librarian") and triage_result["librarian"].get("relevant_files"):
        files = triage_result["librarian"]["relevant_files"]
        files_content = ""
        for i, f in enumerate(files, 1):
            files_content += f"{i}. `{f}`\n"
        comment += _collapsible(
            f"📚 Relevant Files ({len(files)} identified by Librarian)",
            files_content
        )
        comment += "\n"
    
    # Main analysis (clean up branding, use directly)
    cleaned = formatted
    cleaned = cleaned.replace("# 🤖 Gemini Analysis Report", "")
    cleaned = cleaned.replace("# 🤖 Ansieyes Report", "")
    cleaned = cleaned.replace("This analysis was generated by Gemini AI", "")
    cleaned = cleaned.replace("This analysis was generated by Ansieyes AI", "")
    cleaned = cleaned.strip()
    
    comment += "## Analysis\n\n"
    comment += cleaned
    
    comment += _footer()
    
    return comment
```

#### Core Renderer: Security Alert

```python
def format_security_alert(injection_data: dict) -> str:
    """
    Formats prompt injection blocked message.
    
    Args:
        injection_data: Dict with is_injection, risk_level, confidence, detected_patterns
    
    Returns:
        Formatted markdown string
    """
    risk_level = injection_data.get("risk_level", "unknown").upper()
    confidence_percent = int(injection_data.get("confidence", 0) * 100)
    
    comment = _header("Ansieyes Security Alert")
    
    comment += "## 🛑 High-Risk Prompt Injection Detected\n\n"
    comment += "This issue contains patterns attempting to manipulate the AI analysis.\n\n"
    
    comment += _metric_field("🔴", "Risk Level", risk_level)
    comment += _metric_field("📊", "Confidence", f"{confidence_percent}%")
    comment += "\n---\n\n"
    
    # Flagged patterns
    if injection_data.get("detected_patterns"):
        comment += "### 🚨 Flagged Patterns\n\n"
        for i, pattern in enumerate(injection_data['detected_patterns'][:3], 1):
            preview = pattern[:80] + '...' if len(pattern) > 80 else pattern
            comment += f"{i}. `{preview}`\n"
        comment += "\n"
    
    comment += "### 🚫 Action Taken\n\n"
    comment += "**Analysis has been halted for security reasons.**\n\n"
    comment += "If this is a false positive, please rephrase the issue and try again.\n"
    
    comment += _footer()
    
    return comment
```

#### Core Renderer: Duplicate Report

```python
def format_duplicate_report(duplicate_data: dict) -> str:
    """
    Formats duplicate issue detection message.
    
    Args:
        duplicate_data: Dict with is_duplicate, duplicate_of, similarity_score,
                       confidence_score, similarity_reasons
    
    Returns:
        Formatted markdown string
    """
    dup_of = duplicate_data.get('duplicate_of') or {}
    dup_issue_id = dup_of.get('issue_id', 'unknown') if dup_of else 'unknown'
    dup_title = dup_of.get('title', 'Unknown Title') if dup_of else 'Unknown Title'
    
    similarity_percent = int(duplicate_data.get('similarity_score', 0) * 100)
    confidence_percent = int(duplicate_data.get('confidence_score', 0) * 100)
    
    comment = _header("Ansieyes Triage Report")
    
    comment += "## 🔍 Duplicate Issue Detected\n\n"
    comment += f"This issue appears to be a duplicate of **#{dup_issue_id}**: *{dup_title}*\n\n"
    
    comment += _metric_field("📊", "Similarity Score", f"{similarity_percent}%")
    comment += _metric_field("🎯", "Confidence", f"{confidence_percent}%")
    comment += "\n---\n\n"
    
    # Similarity reasons
    if duplicate_data.get('similarity_reasons'):
        reasons_content = ""
        for i, reason in enumerate(duplicate_data['similarity_reasons'][:5], 1):
            reasons_content += f"{i}. {reason}\n"
        comment += _collapsible("🔗 Similarity Reasons", reasons_content, open_by_default=True)
        comment += "\n"
    
    comment += "### 💡 Recommendation\n\n"
    comment += f"Please review issue #{dup_issue_id} and consider closing this as a duplicate.\n"
    
    comment += _footer()
    
    return comment
```

#### Core Renderer: Workflow Analysis

```python
def format_workflow_analysis(analysis: str, workflow_name: str, conclusion: str,
                              failed_jobs: list, workflow_url: str) -> str:
    """
    Formats GitHub Actions workflow analysis.
    Replaces the inline format_workflow_comment() in app.py.
    
    Args:
        analysis: AI-generated analysis text
        workflow_name: Name of the workflow
        conclusion: success/failure/cancelled
        failed_jobs: List of failed job names
        workflow_url: URL to the workflow run
    
    Returns:
        Formatted markdown string
    """
    status_emoji = {"success": "✅", "failure": "❌", "cancelled": "⚠️"}.get(conclusion, "⚠️")
    
    subtitle = f"**Workflow:** {workflow_name} | **Status:** `{conclusion.upper()}`"
    comment = _header(f"{status_emoji} Workflow Analysis", subtitle)
    
    if workflow_url:
        comment += f"[View Workflow Run]({workflow_url})\n\n"
    
    if failed_jobs:
        comment += f"**Failed Jobs:** {', '.join(f'`{j}`' for j in failed_jobs)}\n\n"
    
    comment += "## Analysis\n\n"
    comment += analysis
    
    comment += _footer()
    
    return comment
```

#### Status & Error Messages

```python
def format_processing_message(command: str) -> str:
    """
    Standardized processing/initiated message.
    
    Args:
        command: The command being processed (e.g., "PR Review", "Issue Triage", "Walkthrough")
    
    Returns:
        Formatted markdown string
    """
    return (
        f"## ⏳ Ansieyes {command} Initiated\n\n"
        f"Analyzing... This may take a few moments.\n\n"
        f"Results will be posted here when complete.\n\n"
        f"---\n<sub>Powered by Ansieyes</sub>"
    )


def format_error_message(error_type: str, details: str = "", suggestion: str = "") -> str:
    """
    Standardized error message.
    
    Args:
        error_type: Short error category (e.g., "timeout", "configuration", "analysis")
        details: Error details/message
        suggestion: What the user can do about it
    
    Returns:
        Formatted markdown string
    """
    comment = f"## ⚠️ {error_type.title()} Error\n\n"
    if details:
        comment += f"{details}\n\n"
    if suggestion:
        comment += f"**Suggestion:** {suggestion}\n\n"
    comment += "---\n<sub>Powered by Ansieyes</sub>"
    return comment


def format_unauthorized_message(username: str) -> str:
    """
    Standardized unauthorized access message.
    
    Args:
        username: The GitHub username that attempted the command
    
    Returns:
        Formatted markdown string
    """
    return (
        "## 🚫 Unauthorized\n\n"
        f"@{username}, only repository collaborators with **write** access "
        "or higher are authorized to trigger Ansieyes commands.\n\n"
        "If you believe this is an error, please contact a repository maintainer.\n\n"
        "---\n<sub>Powered by Ansieyes</sub>"
    )


def format_invalid_context_message(command: str, expected: str, actual: str, alternative: str) -> str:
    """
    Wrong command on wrong context message.
    
    Args:
        command: The command used (e.g., "\\ansieyes_triage")
        expected: What it should be used on (e.g., "issues")
        actual: What it was used on (e.g., "pull requests")
        alternative: The correct command (e.g., "\\ansieyes_prreview")
    
    Returns:
        Formatted markdown string
    """
    return (
        f"## ⚠️ Invalid Command\n\n"
        f"`{command}` can only be used on **{expected}**, not {actual}.\n\n"
        f"For {actual}, please use `{alternative}` instead.\n\n"
        "---\n<sub>Powered by Ansieyes</sub>"
    )
```

#### Feature 2 Renderer: JIRA Created

```python
def format_jira_created(jira_key: str, jira_url: str) -> str:
    """
    Formats the JIRA ticket created confirmation message.
    
    Args:
        jira_key: JIRA ticket key (e.g., "PROJ-123")
        jira_url: URL to the JIRA ticket
    
    Returns:
        Formatted markdown string
    """
    return (
        f"## 🎫 JIRA Ticket Created\n\n"
        f"**Ticket:** [{jira_key}]({jira_url})\n\n"
        f"The triage results have been synced to JIRA.\n\n"
        "---\n<sub>Powered by Ansieyes</sub>"
    )
```

#### Feature 3 Renderer: Walkthrough

```python
def format_walkthrough(walkthrough_data: dict) -> str:
    """
    Formats PR walkthrough with Mermaid sequence diagram.
    
    Args:
        walkthrough_data: Dict with keys:
            - change_summary: str
            - file_groups: list of {"group_name": str, "files": [{"name": str, "purpose": str, "changes": str}]}
            - sequence_diagram: str (Mermaid code)
            - impact_analysis: list of str (file paths)
            - pr_title: str (optional)
            - pr_number: int (optional)
    
    Returns:
        Formatted markdown string
    """
    pr_title = walkthrough_data.get('pr_title', '')
    pr_number = walkthrough_data.get('pr_number', '')
    
    subtitle = ""
    if pr_number and pr_title:
        subtitle = f"**PR #{pr_number}** — {pr_title}"
    
    comment = _header("Ansieyes PR Walkthrough", subtitle)
    
    # Change summary
    comment += "## What Changed\n\n"
    comment += walkthrough_data.get('change_summary', 'No summary available.') + "\n\n"
    
    # File groups
    file_groups = walkthrough_data.get('file_groups', [])
    if file_groups:
        comment += "## Change Groups\n\n"
        for group in file_groups:
            group_name = group.get('group_name', 'Other')
            files = group.get('files', [])
            
            table = "| File | What Changed |\n|------|-------------|\n"
            for f in files:
                table += f"| `{f.get('name', '')}` | {f.get('changes', f.get('purpose', ''))} |\n"
            
            comment += f"### {group_name}\n\n{table}\n"
    
    # Sequence diagram
    diagram = walkthrough_data.get('sequence_diagram', '')
    if diagram and 'sequenceDiagram' in diagram.lower().replace(' ', ''):
        comment += "## Interaction Flow\n\n"
        comment += f"```mermaid\n{diagram.strip()}\n```\n\n"
    
    # Impact analysis
    impact = walkthrough_data.get('impact_analysis', [])
    if impact:
        impact_content = "These files were **NOT modified** but may be affected:\n\n"
        for f in impact:
            impact_content += f"- `{f}`\n"
        comment += _collapsible("⚡ Potential Impact (unmodified files)", impact_content)
        comment += "\n"
    
    comment += _footer()
    
    return comment
```

#### Feature 4 Renderer: Auto-Fix Suggestions

```python
def format_fix_suggestions(fix_result: dict) -> str:
    """
    Formats auto-fix suggestions as a GitHub comment.
    
    Args:
        fix_result: Dict with keys:
            - has_fixes: bool
            - fixes: list of dicts, each with:
                - finding: str (description of the issue)
                - severity: str (critical, high)
                - file: str (filename)
                - original_code: str (the problematic code)
                - suggested_fix: str (the fixed code)
                - explanation: str (why this fix is needed)
    
    Returns:
        Formatted markdown string
    """
    fixes = fix_result.get('fixes', [])
    
    if not fixes:
        return ""
    
    comment = _header("Ansieyes Auto-Fix Suggestions")
    comment += f"Found **{len(fixes)}** high-severity issue(s) with suggested fixes.\n\n"
    comment += "---\n\n"
    
    for i, fix in enumerate(fixes, 1):
        sev = _severity_badge(fix.get('severity', 'high'))
        finding = fix.get('finding', 'Unknown issue')
        filename = fix.get('file', 'unknown')
        original = fix.get('original_code', '')
        suggested = fix.get('suggested_fix', '')
        explanation = fix.get('explanation', '')
        
        fix_content = f"**Finding:** {finding}\n"
        fix_content += f"**Severity:** {sev} | **File:** `{filename}`\n\n"
        
        if original and suggested:
            fix_content += "```diff\n"
            for line in original.strip().splitlines():
                fix_content += f"- {line}\n"
            for line in suggested.strip().splitlines():
                fix_content += f"+ {line}\n"
            fix_content += "```\n\n"
        elif suggested:
            fix_content += f"**Suggested code:**\n```\n{suggested.strip()}\n```\n\n"
        
        if explanation:
            fix_content += f"**Why this fix:** {explanation}\n"
        
        # First fix is open by default
        open_default = (i == 1)
        sev_emoji = {'critical': '🔴', 'high': '🟠'}.get(fix.get('severity', '').lower(), '🟡')
        comment += _collapsible(
            f"{sev_emoji} Fix {i}: {finding[:60]} ({filename})",
            fix_content,
            open_by_default=open_default
        )
        comment += "\n"
    
    comment += "---\n"
    comment += "> ⚠️ These are AI-generated suggestions. Please review carefully before applying.\n"
    comment += _footer()
    
    return comment
```

### Migrations in `app.py`

After creating `output_formatter.py`, make these changes in `app.py`:

**Add import (line 19):**
```python
from output_formatter import (
    format_pr_review, format_triage_report, format_workflow_analysis,
    format_processing_message, format_error_message, format_unauthorized_message,
    format_invalid_context_message
)
```

**Replace unauthorized message (lines 273-279):**
```python
# BEFORE:
issue.create_comment(
    "## :no_entry: Unauthorized\n\n"
    f"@{comment_author}, only repository collaborators with **write** "
    "access or higher are authorized to trigger Ansieyes commands.\n\n"
    "If you believe this is an error, please contact a repository maintainer.\n\n"
    "---\n*This is an automated response from Ansieyes.*"
)

# AFTER:
issue.create_comment(format_unauthorized_message(comment_author))
```

**Replace triage invalid context message (lines 564-571):**
```python
# BEFORE:
error_comment = """## ⚠️ Invalid Command
`\\ansieyes_triage` can only be used on **issues**, not pull requests.
For PR reviews, please use `\\ansieyes_prreview` instead.
---
*This is an automated response from Ansieyes.*"""

# AFTER:
error_comment = format_invalid_context_message(
    "\\ansieyes_triage", "issues", "pull requests", "\\ansieyes_prreview"
)
```

**Replace triage processing message (lines 577-581):**
```python
# BEFORE:
processing_comment = issue.create_comment(
    "## Ansieyes Issue Triage has been Initiated\n\n"
    "This may take a few minutes. Results will be posted here.\n\n"
    "---\n*Powered by Ansieyes using AI-Issue-Triage*"
)

# AFTER:
processing_comment = issue.create_comment(format_processing_message("Issue Triage"))
```

**Replace timeout error (lines 602-612):**
```python
# BEFORE (the big multiline string):
issue.create_comment("## ⚠️ Timeout Error\n\n" ... )

# AFTER:
issue.create_comment(format_error_message(
    "timeout",
    "Repository clone took too long (>5 minutes).",
    "This might be due to a very large repository or network issues. Try again later."
))
```

**Replace config error (lines 617-621):**
```python
# BEFORE:
issue.create_comment("## ⚠️ Configuration Error\n\n" ...)

# AFTER:
issue.create_comment(format_error_message(
    "configuration",
    f"Could not clone repository: {str(e)}"
))
```

**Replace triage comment formatting (line 685):**
```python
# BEFORE:
comment_body = issue_triager.format_triage_comment(triage_result)

# AFTER:
comment_body = format_triage_report(triage_result)
```

**Replace workflow analysis formatting (line 509):**
```python
# BEFORE:
comment_body = format_workflow_comment(analysis, workflow_name, conclusion, failed_jobs, workflow_run.get('html_url', ''))

# AFTER:
comment_body = format_workflow_analysis(analysis, workflow_name, conclusion, failed_jobs, workflow_run.get('html_url', ''))
```

**Delete `format_workflow_comment()` function (lines 519-537).**

**Replace PR review invalid context (lines 871-878):**
```python
# BEFORE:
error_comment = """## ⚠️ Invalid Command
`\\ansieyes_prreview` can only be used on **pull requests**, not regular issues.
For issue triage, please use `\\ansieyes_triage` instead.
---
*This is an automated response from Ansieyes.*"""

# AFTER:
error_comment = format_invalid_context_message(
    "\\ansieyes_prreview", "pull requests", "regular issues", "\\ansieyes_triage"
)
```

**Replace PR review processing message (lines 887-891):**
```python
# BEFORE:
processing_comment = issue.create_comment(
    "## Ansieyes PR Review Initiated\n\n" ...
)

# AFTER:
processing_comment = issue.create_comment(format_processing_message("PR Review"))
```

**Replace PR review error fallback (lines 928-931):**
```python
# BEFORE:
issue.create_comment(review_text or 
    "## ⚠️ Review Failed\n\n" ...
)

# AFTER:
issue.create_comment(review_text or format_error_message("review", "Could not generate review."))
```

**Wrap successful PR review in formatter (line 936):**
```python
# BEFORE:
issue.create_comment(review_text)

# AFTER:
formatted_review = format_pr_review(
    review_text=review_text,
    file_changes=file_changes,
    metadata={'pr_title': title, 'pr_number': issue_number}
)
issue.create_comment(formatted_review)
```

### Migrations in `issue_triager.py`

Make `format_triage_comment()` (line 617) delegate to `output_formatter`:

```python
def format_triage_comment(self, triage_result: Dict) -> str:
    """
    Format triage results as a GitHub comment.
    Delegates to output_formatter for consistent formatting.
    """
    from output_formatter import format_triage_report
    return format_triage_report(triage_result)
```

### Migrations in `pr_reviewer.py`

Make `format_review_summary()` (line 118) delegate to `output_formatter`:

```python
def format_review_summary(self, review_text: str) -> str:
    """
    Format review. Delegates to output_formatter.
    """
    from output_formatter import format_pr_review
    return format_pr_review(review_text)
```

---

## Phase 2a: JIRA Integration

### New File: `jira_connector.py`

Create at `/Users/jeevans/Desktop/Ansieyes/Ansieyes/jira_connector.py`

```python
#!/usr/bin/env python3
"""
JIRA Integration for Ansieyes
Creates JIRA tickets from triage results.
"""
import os
import re
import logging
from typing import Dict, Optional

logger = logging.getLogger(__name__)


class JiraConnector:
    """Handle JIRA ticket creation from Ansieyes triage results."""

    def __init__(self):
        """
        Initialize JIRA connector from environment variables.
        
        Required env vars (when JIRA_ENABLED=true):
            JIRA_URL: Atlassian Cloud URL (e.g., https://yourname.atlassian.net)
            JIRA_EMAIL: Your Atlassian account email
            JIRA_API_TOKEN: API token from id.atlassian.com/manage-profile/security/api-tokens
            JIRA_PROJECT_KEY: Project key (e.g., PROJ, ANS)
        """
        self.enabled = os.getenv('JIRA_ENABLED', 'false').lower() == 'true'
        self.url = os.getenv('JIRA_URL', '')
        self.email = os.getenv('JIRA_EMAIL', '')
        self.api_token = os.getenv('JIRA_API_TOKEN', '')
        self.project_key = os.getenv('JIRA_PROJECT_KEY', '')
        self.client = None
        
        if self.enabled:
            if not all([self.url, self.email, self.api_token, self.project_key]):
                logger.warning(
                    "JIRA_ENABLED=true but missing required vars. "
                    "Need: JIRA_URL, JIRA_EMAIL, JIRA_API_TOKEN, JIRA_PROJECT_KEY. "
                    "JIRA integration disabled."
                )
                self.enabled = False
                return
            
            try:
                from jira import JIRA
                self.client = JIRA(
                    server=self.url,
                    basic_auth=(self.email, self.api_token)
                )
                logger.info(f"JIRA connector initialized: {self.url} (project: {self.project_key})")
            except ImportError:
                logger.error("'jira' package not installed. Run: pip install jira>=3.5.0")
                self.enabled = False
            except Exception as e:
                logger.error(f"Failed to initialize JIRA client: {e}")
                self.enabled = False

    def is_enabled(self) -> bool:
        """Check if JIRA integration is active and ready."""
        return self.enabled and self.client is not None

    def create_ticket(self, triage_result: Dict, issue_url: str,
                      issue_number: int, repo_name: str) -> Dict:
        """
        Create a JIRA ticket from triage results.
        
        Args:
            triage_result: Full triage result dict from IssueTriager
            issue_url: GitHub issue URL
            issue_number: GitHub issue number
            repo_name: Full repo name (owner/repo)
        
        Returns:
            {'key': 'PROJ-123', 'url': 'https://...', 'error': None} on success
            {'key': None, 'url': None, 'error': 'message'} on failure
        """
        if not self.is_enabled():
            return {'key': None, 'url': None, 'error': 'JIRA not enabled'}
        
        try:
            # Extract type and severity from surgeon output
            issue_type, severity = self._extract_classification(triage_result)
            
            # Extract title from triage result or use generic
            title = triage_result.get('title', f'GitHub Issue #{issue_number}')
            
            # Map to JIRA types
            jira_issue_type = self._map_type_to_jira(issue_type)
            jira_priority = self._map_severity_to_jira(severity)
            
            # Build description
            description = self._build_description(triage_result, issue_url, repo_name)
            
            # Create the ticket
            issue_dict = {
                'project': {'key': self.project_key},
                'summary': f'[GitHub #{issue_number}] {title}',
                'description': description,
                'issuetype': {'name': jira_issue_type},
            }
            
            # Try to set priority (may fail if project doesn't support it)
            try:
                issue_dict['priority'] = {'name': jira_priority}
            except Exception:
                pass
            
            new_issue = self.client.create_issue(fields=issue_dict)
            
            ticket_url = f"{self.url}/browse/{new_issue.key}"
            logger.info(f"Created JIRA ticket: {new_issue.key} ({ticket_url})")
            
            return {
                'key': new_issue.key,
                'url': ticket_url,
                'error': None
            }
            
        except Exception as e:
            logger.error(f"Failed to create JIRA ticket: {e}")
            return {'key': None, 'url': None, 'error': str(e)}

    def _extract_classification(self, triage_result: Dict) -> tuple:
        """
        Extract type and severity from triage results.
        Uses regex on surgeon formatted_output (same patterns as app.py lines 770-777).
        
        Returns:
            (issue_type: str, severity: str)
        """
        issue_type = "Task"
        severity = "Medium"
        
        surgeon = triage_result.get("surgeon", {})
        if surgeon and "formatted_output" in surgeon:
            formatted = surgeon["formatted_output"]
            
            type_match = re.search(r'\*\*Type:\*\*\s+`([^`]+)`', formatted)
            if type_match:
                issue_type = type_match.group(1)
            
            severity_match = re.search(r'\*\*Severity:\*\*\s+`([^`]+)`', formatted)
            if severity_match:
                severity = severity_match.group(1)
        
        return issue_type, severity

    def _map_type_to_jira(self, ansieyes_type: str) -> str:
        """
        Map Ansieyes issue type to JIRA issue type name.
        
        Mapping:
            Bug -> Bug
            Enhancement -> Story
            Feature Request -> Story
            Other/Unknown -> Task
        """
        type_map = {
            'bug': 'Bug',
            'enhancement': 'Story',
            'feature_request': 'Story',
            'feature request': 'Story',
        }
        return type_map.get(ansieyes_type.lower(), 'Task')

    def _map_severity_to_jira(self, ansieyes_severity: str) -> str:
        """
        Map Ansieyes severity to JIRA priority name.
        
        Mapping:
            Critical -> Highest
            High -> High
            Medium -> Medium
            Low -> Low
            Unknown -> Medium
        """
        priority_map = {
            'critical': 'Highest',
            'high': 'High',
            'medium': 'Medium',
            'low': 'Low',
        }
        return priority_map.get(ansieyes_severity.lower(), 'Medium')

    def _build_description(self, triage_result: Dict, issue_url: str, repo_name: str) -> str:
        """
        Build JIRA ticket description from triage results.
        Uses Jira wiki markup (not markdown).
        """
        issue_type, severity = self._extract_classification(triage_result)
        
        desc = f"h2. GitHub Issue\n\n"
        desc += f"*Source:* [{repo_name}|{issue_url}]\n\n"
        
        desc += "h2. AI Triage Classification\n\n"
        desc += f"* *Type:* {issue_type}\n"
        desc += f"* *Severity:* {severity}\n"
        desc += f"* *Repository:* {repo_name}\n\n"
        
        # Add surgeon summary (first 1000 chars)
        surgeon = triage_result.get("surgeon", {})
        if surgeon and "formatted_output" in surgeon:
            summary = surgeon["formatted_output"][:1000]
            # Strip markdown for Jira wiki
            summary = summary.replace('#', '').replace('**', '*').replace('`', '{{').replace('`', '}}')
            desc += "h2. Analysis Summary\n\n"
            desc += summary + "\n\n"
        
        desc += "----\n_Auto-created by Ansieyes_"
        
        return desc
```

### Changes to `app.py`

**Add import (near line 19):**
```python
from jira_connector import JiraConnector
```

**Add initialization (near line 52, after issue_triager):**
```python
# Initialize JIRA Connector
jira_connector = JiraConnector()
```

**Add JIRA integration in `handle_triage_mention()`, AFTER labels are applied (after line 833):**
```python
        # JIRA Integration: Create ticket if enabled and not blocked/duplicate
        if jira_connector.is_enabled() and not is_blocked and not is_duplicate:
            logger.info("JIRA integration: Attempting to create ticket...")
            try:
                # Get issue title for JIRA ticket
                jira_triage_result = dict(triage_result)
                jira_triage_result['title'] = title
                
                jira_result = jira_connector.create_ticket(
                    triage_result=jira_triage_result,
                    issue_url=issue_data.get('html_url', ''),
                    issue_number=issue_number,
                    repo_name=repo_full_name
                )
                
                if jira_result.get('key'):
                    from output_formatter import format_jira_created
                    jira_comment = format_jira_created(
                        jira_result['key'], jira_result['url']
                    )
                    issue.create_comment(jira_comment)
                    logger.info(f"JIRA ticket created: {jira_result['key']}")
                    
                    # Add JIRA key as label
                    try:
                        jira_label = jira_result['key']
                        existing_repo_labels = {l.name for l in repo.get_labels()}
                        if jira_label not in existing_repo_labels:
                            repo.create_label(name=jira_label, color='0052CC')
                        issue.add_to_labels(jira_label)
                    except Exception as e:
                        logger.warning(f"Could not add JIRA label: {e}")
                else:
                    logger.warning(f"JIRA ticket creation failed: {jira_result.get('error')}")
                    
            except Exception as e:
                logger.error(f"JIRA integration error: {e}")
                # Never let JIRA failure affect triage
```

### Environment Variables

Add to `env_example.txt`:
```
# --- JIRA Integration (optional, disabled by default) ---
# Set to 'true' to auto-create JIRA tickets after issue triage
JIRA_ENABLED=false
# Your Atlassian Cloud URL
JIRA_URL=https://yourname.atlassian.net
# Your Atlassian account email
JIRA_EMAIL=your-email@example.com
# API token from: https://id.atlassian.com/manage-profile/security/api-tokens
JIRA_API_TOKEN=your_jira_api_token
# JIRA project key (e.g., ANS, PROJ)
JIRA_PROJECT_KEY=PROJ
```

### Dependency

Add to `requirements.txt`:
```
# JIRA Integration
jira>=3.5.0
```

---

## Phase 2b: PR Walkthrough + Sequence Diagrams

### New Command: `\ansieyes_walkthrough`

Exact match, PRs only. Generates visual walkthrough with Mermaid sequence diagram.

### New File: `walkthrough_generator.py`

Create at `/Users/jeevans/Desktop/Ansieyes/Ansieyes/walkthrough_generator.py`

```python
#!/usr/bin/env python3
"""
PR Walkthrough Generator with Sequence Diagrams
Uses Gemini API directly (not AI-Issue-Triage CLI)
"""
import json
import logging
import os
from typing import Dict, List, Optional

logger = logging.getLogger(__name__)


class WalkthroughGenerator:
    """Generate PR walkthroughs with Mermaid sequence diagrams."""

    def __init__(self, api_key: Optional[str] = None):
        """
        Initialize with Gemini API key.
        Uses google-generativeai directly.
        """
        self.api_key = api_key or os.getenv("GEMINI_API_KEY")
        self.model = None
        
        if self.api_key:
            try:
                import google.generativeai as genai
                genai.configure(api_key=self.api_key)
                self.model = genai.GenerativeModel('gemini-2.0-flash-001')
                logger.info("WalkthroughGenerator initialized with Gemini")
            except Exception as e:
                logger.error(f"Failed to initialize Gemini for walkthrough: {e}")
        else:
            logger.warning("No Gemini API key for WalkthroughGenerator")

    def generate_walkthrough(self, title: str, body: str,
                              file_changes: List[Dict],
                              repo_url: Optional[str] = None) -> Dict:
        """
        Generate structured walkthrough data for a PR.
        
        Args:
            title: PR title
            body: PR description
            file_changes: List of file change dicts with filename, status, additions, deletions, patch
            repo_url: Repository URL for context
        
        Returns:
            Dict with keys: change_summary, file_groups, sequence_diagram, impact_analysis
            Or: {'error': str} on failure
        """
        if not self.model:
            return {'error': 'Gemini model not available'}
        
        try:
            prompt = self._build_prompt(title, body, file_changes)
            
            response = self.model.generate_content(
                prompt,
                generation_config={
                    'temperature': 0.3,
                    'max_output_tokens': 4096,
                }
            )
            
            response_text = response.text.strip()
            
            # Try to parse as JSON
            # Strip markdown code fence if present
            if response_text.startswith('```'):
                lines = response_text.split('\n')
                # Remove first and last lines (```json and ```)
                response_text = '\n'.join(lines[1:-1] if lines[-1].strip() == '```' else lines[1:])
            
            try:
                result = json.loads(response_text)
            except json.JSONDecodeError:
                logger.warning("Failed to parse walkthrough as JSON, using raw text")
                result = {
                    'change_summary': response_text[:500],
                    'file_groups': [],
                    'sequence_diagram': '',
                    'impact_analysis': []
                }
            
            # Validate and sanitize Mermaid diagram
            if result.get('sequence_diagram'):
                result['sequence_diagram'] = self._validate_mermaid(result['sequence_diagram'])
            
            return result
            
        except Exception as e:
            logger.error(f"Walkthrough generation failed: {e}")
            return {'error': str(e)}

    def _build_prompt(self, title: str, body: str, file_changes: List[Dict]) -> str:
        """
        Build the Gemini prompt for walkthrough generation.
        Requests structured JSON output with Mermaid sequence diagram.
        """
        # Build file changes summary
        changes_text = ""
        for fc in file_changes[:30]:  # Limit to 30 files
            fname = fc.get('filename', 'unknown')
            status = fc.get('status', 'modified')
            adds = fc.get('additions', 0)
            dels = fc.get('deletions', 0)
            patch = fc.get('patch', '')
            
            changes_text += f"\n### {fname} ({status}, +{adds} -{dels})\n"
            if patch:
                # Truncate large patches
                patch_lines = patch.split('\n')
                if len(patch_lines) > 80:
                    changes_text += '\n'.join(patch_lines[:80])
                    changes_text += f"\n... ({len(patch_lines) - 80} more lines truncated)\n"
                else:
                    changes_text += patch + "\n"
        
        prompt = f"""You are an expert code analyst. Analyze this pull request and return ONLY valid JSON (no markdown fences, no extra text).

## Pull Request
**Title:** {title}
**Description:** {body or 'No description provided'}

## File Changes
{changes_text}

## Required JSON Output

Return this exact JSON structure:

{{
  "change_summary": "2-3 sentence plain English summary of what this PR does and why",
  "file_groups": [
    {{
      "group_name": "Logical grouping name (e.g., 'API Layer', 'Authentication', 'Tests')",
      "files": [
        {{
          "name": "path/to/file.py",
          "changes": "One-line description of what changed in this file"
        }}
      ]
    }}
  ],
  "sequence_diagram": "Valid Mermaid sequenceDiagram code showing the runtime interaction flow between components affected by this PR. Include participant aliases. Focus on the main flow, not every edge case. If the changes don't have a clear interaction flow, show the data/control flow through the changed files.",
  "impact_analysis": ["path/to/potentially-affected-file.py that was NOT modified but could be affected by these changes"]
}}

IMPORTANT:
- The sequence_diagram MUST start with 'sequenceDiagram' on the first line
- Use short participant aliases (e.g., 'participant C as Controller')
- Group files logically by feature/area, not by file type
- impact_analysis should list files NOT in the PR that might need attention
- Return ONLY the JSON, nothing else
"""
        return prompt

    def _validate_mermaid(self, mermaid_code: str) -> str:
        """
        Basic validation and sanitization of Mermaid diagram code.
        
        Returns sanitized code, or empty string if invalid.
        """
        if not mermaid_code or not isinstance(mermaid_code, str):
            return ''
        
        code = mermaid_code.strip()
        
        # Strip markdown code fences if present
        if code.startswith('```'):
            lines = code.split('\n')
            code = '\n'.join(lines[1:-1] if lines[-1].strip() == '```' else lines[1:])
            code = code.strip()
        
        # Must contain sequenceDiagram keyword
        if 'sequencediagram' not in code.lower().replace(' ', '').replace('\n', ''):
            logger.warning("Mermaid code missing 'sequenceDiagram' keyword")
            return ''
        
        # Ensure it starts with sequenceDiagram
        if not code.startswith('sequenceDiagram'):
            # Try to find and extract from the keyword
            idx = code.lower().find('sequencediagram')
            if idx >= 0:
                code = 'sequenceDiagram' + code[idx + len('sequenceDiagram'):]
            else:
                return ''
        
        return code
```

### Changes to `app.py`

**Add import (near line 19):**
```python
from walkthrough_generator import WalkthroughGenerator
```

**Add initialization (near line 52):**
```python
# Initialize Walkthrough Generator
walkthrough_gen = WalkthroughGenerator(GEMINI_API_KEY)
```

**Add to bot command set (line 250):**
```python
# BEFORE:
is_bot_command = comment_body in ('\\ansieyes_triage', '\\ansieyes_prreview')

# AFTER:
is_bot_command = comment_body in ('\\ansieyes_triage', '\\ansieyes_prreview', '\\ansieyes_walkthrough')
```

**Add elif routing (after line 298, before the `return` statement):**
```python
            elif comment_body == '\\ansieyes_walkthrough':
                logger.info("Detected exact \\ansieyes_walkthrough mention")
                try:
                    handle_walkthrough_mention(payload, installation_id)
                except Exception as e:
                    logger.error(f"Error processing walkthrough mention: {e}")
                    return jsonify({"error": str(e)}), 500
```

**Add new handler function (after `handle_pr_review_mention()`, before `if __name__`):**

```python
def handle_walkthrough_mention(payload, installation_id):
    """Handle \\ansieyes_walkthrough mention in PR comments"""
    issue_data = payload.get('issue', {})
    repo_full_name = payload.get('repository', {}).get('full_name')
    issue_number = issue_data.get('number')
    
    is_pull_request = 'pull_request' in issue_data
    
    logger.info(f"Walkthrough mention on {'PR' if is_pull_request else 'issue'} #{issue_number} in {repo_full_name}")
    
    github_client = get_github_client(installation_id)
    if not github_client:
        logger.error("Failed to create GitHub client")
        return
    
    try:
        repo = github_client.get_repo(repo_full_name)
        issue = repo.get_issue(issue_number)
        
        # Validation: walkthrough only works on PRs
        if not is_pull_request:
            issue.create_comment(format_invalid_context_message(
                "\\ansieyes_walkthrough", "pull requests", "regular issues", "\\ansieyes_triage"
            ))
            logger.warning(f"\\ansieyes_walkthrough used on issue #{issue_number}")
            return
        
        pr = repo.get_pull(issue_number)
        
        # Post processing message
        processing_comment = issue.create_comment(format_processing_message("PR Walkthrough"))
        
        # Get PR details
        title = pr.title
        body = pr.body or ""
        
        # Get file changes
        files = pr.get_files()
        file_changes = []
        for file in files:
            file_info = {
                'filename': file.filename,
                'status': file.status,
                'additions': file.additions,
                'deletions': file.deletions,
                'changes': file.changes,
                'patch': file.patch if hasattr(file, 'patch') else None
            }
            file_changes.append(file_info)
        
        logger.info(f"Found {len(file_changes)} changed files for walkthrough")
        
        repo_url = payload.get('repository', {}).get('html_url', '') or repo.html_url
        
        # Generate walkthrough
        walkthrough_data = walkthrough_gen.generate_walkthrough(
            title=title,
            body=body,
            file_changes=file_changes,
            repo_url=repo_url
        )
        
        if walkthrough_data.get('error'):
            issue.create_comment(format_error_message(
                "walkthrough", walkthrough_data['error']
            ))
            try:
                processing_comment.delete()
            except:
                pass
            return
        
        # Add PR metadata for the formatter
        walkthrough_data['pr_title'] = title
        walkthrough_data['pr_number'] = issue_number
        
        # Format and post
        from output_formatter import format_walkthrough
        comment_body = format_walkthrough(walkthrough_data)
        issue.create_comment(comment_body)
        
        # Delete processing comment
        try:
            processing_comment.delete()
        except:
            pass
        
        logger.info(f"Walkthrough completed for PR #{issue_number}")
        
    except Exception as e:
        logger.error(f"Error handling walkthrough mention: {e}")
        import traceback
        traceback.print_exc()
```

### No new dependencies or env vars needed.

---

## Phase 3: Auto-Fix Agent

### How It Works
After `\ansieyes_prreview` posts a review, the auto-fixer:
1. Parses the review text for HIGH/CRITICAL severity findings
2. For each finding (max 3), sends the finding + relevant file patch to Gemini
3. Gemini returns a concrete code fix
4. Posts a SEPARATE follow-up comment with diff-style fix suggestions
5. **Never commits, never auto-merges** — suggestion only (Phase 1)

### New File: `auto_fixer.py`

Create at `/Users/jeevans/Desktop/Ansieyes/Ansieyes/auto_fixer.py`

```python
#!/usr/bin/env python3
"""
Auto-Fix Agent for Ansieyes
Analyzes PR review findings and generates code fix suggestions.
Phase 1: Suggestion-only (no commits/PRs)
"""
import json
import logging
import os
import re
from typing import Dict, List, Optional

logger = logging.getLogger(__name__)


class AutoFixer:
    """Analyze PR reviews and generate fix suggestions for high-severity findings."""

    def __init__(self, api_key: Optional[str] = None):
        """
        Initialize with Gemini API key.
        
        Env vars:
            AUTOFIX_ENABLED: 'true'/'false' (default: 'true')
            AUTOFIX_MIN_SEVERITY: 'critical'/'high'/'medium' (default: 'high')
        """
        self.api_key = api_key or os.getenv("GEMINI_API_KEY")
        self.enabled = os.getenv('AUTOFIX_ENABLED', 'true').lower() == 'true'
        self.min_severity = os.getenv('AUTOFIX_MIN_SEVERITY', 'high').lower()
        self.model = None
        self.max_fixes = 3
        
        if self.api_key and self.enabled:
            try:
                import google.generativeai as genai
                genai.configure(api_key=self.api_key)
                self.model = genai.GenerativeModel('gemini-2.0-flash-001')
                logger.info(f"AutoFixer initialized (min severity: {self.min_severity})")
            except Exception as e:
                logger.error(f"Failed to initialize Gemini for auto-fix: {e}")

    def is_enabled(self) -> bool:
        """Check if auto-fix is active."""
        return self.enabled and self.model is not None

    def analyze_for_fixes(self, review_text: str, file_changes: List[Dict],
                           repo_url: Optional[str] = None) -> Dict:
        """
        Analyze PR review output and generate fix suggestions for high-severity findings.
        
        Args:
            review_text: The AI review text (markdown)
            file_changes: List of file change dicts with filename, patch
            repo_url: Repository URL for context
        
        Returns:
            {
                'has_fixes': bool,
                'fixes': [
                    {
                        'finding': str,
                        'severity': str,
                        'file': str,
                        'original_code': str,
                        'suggested_fix': str,
                        'explanation': str
                    }
                ]
            }
        """
        if not self.is_enabled():
            return {'has_fixes': False, 'fixes': []}
        
        # Step 1: Extract high-severity findings
        findings = self._extract_high_severity_findings(review_text)
        
        if not findings:
            logger.info("No high-severity findings to fix")
            return {'has_fixes': False, 'fixes': []}
        
        logger.info(f"Found {len(findings)} high-severity findings, generating fixes...")
        
        # Step 2: Generate fixes (max 3)
        fixes = []
        file_patches = {fc.get('filename', ''): fc.get('patch', '') for fc in file_changes}
        
        for finding in findings[:self.max_fixes]:
            try:
                filename = finding.get('file', '')
                patch = file_patches.get(filename, '')
                
                fix = self._generate_fix(
                    finding=finding.get('description', ''),
                    file_patch=patch,
                    filename=filename
                )
                
                if fix:
                    fix['severity'] = finding.get('severity', 'high')
                    fix['finding'] = finding.get('description', '')
                    fix['file'] = filename
                    fixes.append(fix)
                    
            except Exception as e:
                logger.warning(f"Failed to generate fix for finding: {e}")
                continue
        
        return {
            'has_fixes': len(fixes) > 0,
            'fixes': fixes
        }

    def _extract_high_severity_findings(self, review_text: str) -> List[Dict]:
        """
        Parse review text to find high/critical severity findings.
        
        Uses multiple extraction strategies:
        1. Regex for severity badges from output_formatter (🔴, 🟠)
        2. Markdown headers with severity keywords
        3. Keyword-based detection
        
        Returns:
            List of {'description': str, 'severity': str, 'file': str}
        """
        findings = []
        severity_threshold = ['critical', 'high']
        if self.min_severity == 'medium':
            severity_threshold.append('medium')
        
        # Strategy 1: Look for severity emoji badges
        # Pattern: 🔴 `CRITICAL` or 🟠 `HIGH` followed by description
        badge_patterns = [
            (r'🔴\s*`?CRITICAL`?\s*[:\-]?\s*(.+?)(?=\n🔴|\n🟠|\n🟡|\n#{2,}|\Z)', 'critical'),
            (r'🟠\s*`?HIGH`?\s*[:\-]?\s*(.+?)(?=\n🔴|\n🟠|\n🟡|\n#{2,}|\Z)', 'high'),
        ]
        if 'medium' in severity_threshold:
            badge_patterns.append(
                (r'🟡\s*`?MEDIUM`?\s*[:\-]?\s*(.+?)(?=\n🔴|\n🟠|\n🟡|\n#{2,}|\Z)', 'medium')
            )
        
        for pattern, severity in badge_patterns:
            matches = re.finditer(pattern, review_text, re.DOTALL | re.IGNORECASE)
            for match in matches:
                desc = match.group(1).strip()[:200]
                # Try to extract file reference
                file_match = re.search(r'`([^`]+\.\w+)`', desc)
                filename = file_match.group(1) if file_match else ''
                findings.append({'description': desc, 'severity': severity, 'file': filename})
        
        # Strategy 2: Look for markdown headers with severity
        header_patterns = [
            (r'###?\s*(?:🔴\s*)?(?:Critical|CRITICAL)[:\s]+(.+?)(?=\n###?\s|\Z)', 'critical'),
            (r'###?\s*(?:🟠\s*)?(?:High|HIGH)[:\s]+(.+?)(?=\n###?\s|\Z)', 'high'),
        ]
        
        for pattern, severity in header_patterns:
            matches = re.finditer(pattern, review_text, re.DOTALL | re.IGNORECASE)
            for match in matches:
                desc = match.group(1).strip()[:200]
                file_match = re.search(r'`([^`]+\.\w+)`', desc)
                filename = file_match.group(1) if file_match else ''
                # Avoid duplicates
                if not any(f['description'][:50] == desc[:50] for f in findings):
                    findings.append({'description': desc, 'severity': severity, 'file': filename})
        
        # Strategy 3: Keyword-based fallback
        if not findings:
            keywords = {
                'critical': ['security vulnerability', 'sql injection', 'xss', 'hardcoded secret',
                            'hardcoded password', 'hardcoded api key', 'remote code execution'],
                'high': ['null pointer', 'null check', 'missing error handling', 'race condition',
                        'memory leak', 'unhandled exception', 'authentication bypass'],
            }
            
            for severity, kws in keywords.items():
                if severity not in severity_threshold:
                    continue
                for kw in kws:
                    if kw in review_text.lower():
                        # Extract surrounding context
                        idx = review_text.lower().index(kw)
                        start = max(0, idx - 50)
                        end = min(len(review_text), idx + 150)
                        context = review_text[start:end].strip()
                        
                        file_match = re.search(r'`([^`]+\.\w+)`', context)
                        filename = file_match.group(1) if file_match else ''
                        
                        if not any(f['description'][:30] == context[:30] for f in findings):
                            findings.append({'description': context, 'severity': severity, 'file': filename})
        
        return findings[:self.max_fixes]

    def _generate_fix(self, finding: str, file_patch: str, filename: str) -> Optional[Dict]:
        """
        Generate a concrete code fix for a single finding using Gemini.
        
        Args:
            finding: Description of the issue
            file_patch: The diff/patch of the file
            filename: Name of the file
        
        Returns:
            {'original_code': str, 'suggested_fix': str, 'explanation': str}
            or None on failure
        """
        if not self.model:
            return None
        
        prompt = f"""You are an expert code fixer. A code review found this issue:

**Finding:** {finding}
**File:** {filename}

**File Patch (diff):**
```
{file_patch[:3000] if file_patch else 'No patch available'}
```

Generate a fix. Return ONLY valid JSON (no markdown fences):

{{
  "original_code": "the exact problematic code lines (just the relevant 1-5 lines)",
  "suggested_fix": "the fixed version of those exact lines",
  "explanation": "One sentence explaining why this fix addresses the issue"
}}

IMPORTANT:
- Keep the fix minimal - only change what's necessary
- The original_code and suggested_fix should be actual code, not descriptions
- If you can't determine the exact code from the patch, provide your best approximation
- Return ONLY the JSON, nothing else
"""
        
        try:
            response = self.model.generate_content(
                prompt,
                generation_config={
                    'temperature': 0.2,
                    'max_output_tokens': 1024,
                }
            )
            
            response_text = response.text.strip()
            
            # Strip markdown code fences
            if response_text.startswith('```'):
                lines = response_text.split('\n')
                response_text = '\n'.join(lines[1:-1] if lines[-1].strip() == '```' else lines[1:])
            
            result = json.loads(response_text)
            
            if result.get('original_code') and result.get('suggested_fix'):
                return result
            else:
                logger.warning("Fix generation returned incomplete data")
                return None
                
        except json.JSONDecodeError as e:
            logger.warning(f"Failed to parse fix JSON: {e}")
            return None
        except Exception as e:
            logger.error(f"Fix generation failed: {e}")
            return None
```

### Changes to `app.py`

**Add import (near line 19):**
```python
from auto_fixer import AutoFixer
```

**Add initialization (near line 52):**
```python
# Initialize Auto-Fixer
auto_fixer = AutoFixer(GEMINI_API_KEY)
```

**Add auto-fix to `handle_pr_review_mention()`, AFTER posting review (after line 936):**
```python
        # Auto-fix: Analyze review for high-severity findings
        if auto_fixer.is_enabled() and review_text and not review_text.startswith("❌"):
            try:
                logger.info("Running auto-fix analysis...")
                fix_result = auto_fixer.analyze_for_fixes(
                    review_text=review_text,
                    file_changes=file_changes,
                    repo_url=repo_url
                )
                if fix_result.get('has_fixes'):
                    from output_formatter import format_fix_suggestions
                    fix_comment = format_fix_suggestions(fix_result)
                    if fix_comment:
                        issue.create_comment(fix_comment)
                        logger.info(f"Posted {len(fix_result['fixes'])} fix suggestions")
                else:
                    logger.info("No high-severity findings for auto-fix")
            except Exception as e:
                logger.error(f"Auto-fix analysis error: {e}")
                # Never let auto-fix failure affect the review
```

**Add same auto-fix block to the bug-fixed `review_pr()`, after posting review:**
```python
        # Post review as PR comment
        pr.create_issue_comment(review_text)
        logger.info("Posted review comment")
        
        # Auto-fix: Analyze for high-severity findings
        if auto_fixer.is_enabled() and review_text and not review_text.startswith("❌"):
            try:
                fix_result = auto_fixer.analyze_for_fixes(
                    review_text=review_text,
                    file_changes=file_changes,
                    repo_url=repo_url
                )
                if fix_result.get('has_fixes'):
                    from output_formatter import format_fix_suggestions
                    fix_comment = format_fix_suggestions(fix_result)
                    if fix_comment:
                        pr.create_issue_comment(fix_comment)
                        logger.info(f"Posted {len(fix_result['fixes'])} auto-fix suggestions")
            except Exception as e:
                logger.error(f"Auto-fix error: {e}")
```

### Environment Variables

Add to `env_example.txt`:
```
# --- Auto-Fix Agent (optional) ---
# Set to 'false' to disable auto-fix suggestions after PR reviews
AUTOFIX_ENABLED=true
# Minimum severity to trigger fix suggestions: critical, high, medium
AUTOFIX_MIN_SEVERITY=high
```

---

## Phase 4: Documentation & Config Updates

### `requirements.txt` - Add:
```
# JIRA Integration
jira>=3.5.0
```

### `env_example.txt` - Add all new env vars (JIRA + AUTOFIX sections shown above)

### `README.md` - Update commands section:
```markdown
## Usage

Comment on GitHub:
- **`\ansieyes_prreview`** - Review a PR (+ auto-fix suggestions for critical issues)
- **`\ansieyes_triage`** - Analyze an issue (+ auto JIRA ticket creation)
- **`\ansieyes_walkthrough`** - Visual PR walkthrough with sequence diagrams

**Important**: Exact match only, no extra text!
```

### `GUIDE.md` - Update command table:
```markdown
| Command | Use On | What It Does | Time |
|---------|--------|--------------|------|
| `\ansieyes_prreview` | Pull Requests ONLY | AI code review + auto-fix | 10-30s |
| `\ansieyes_triage` | Issues ONLY | Two-pass analysis + labeling + JIRA | 30-70s |
| `\ansieyes_walkthrough` | Pull Requests ONLY | Visual walkthrough + sequence diagram | 15-30s |
```

Add new sections for:
- JIRA Integration setup (env vars, how to get API token)
- Walkthrough feature description
- Auto-fix feature description

### `triage.config.example.json` - Add JIRA section:
```json
{
  "jira": {
    "enabled": false,
    "url": "https://yourname.atlassian.net",
    "project_key": "PROJ"
  }
}
```

### `context.md` - Update with:
- New files: `output_formatter.py`, `jira_connector.py`, `walkthrough_generator.py`, `auto_fixer.py`
- New commands: `\ansieyes_walkthrough`
- New features: JIRA integration, auto-fix, structured output
- Updated architecture diagram

---

## New File Structure

```
Ansieyes/
+-- app.py                        # MODIFIED: bug fix, new command routing, formatter migration, 
|                                 #           JIRA integration, auto-fix integration
+-- pr_reviewer.py                # MODIFIED: delegate format_review_summary to output_formatter
+-- issue_triager.py              # MODIFIED: delegate format_triage_comment to output_formatter
+-- output_formatter.py           # NEW: centralized output templates (~400 lines)
+-- jira_connector.py             # NEW: JIRA ticket creation (~180 lines)
+-- walkthrough_generator.py      # NEW: PR walkthrough + Mermaid diagrams (~220 lines)
+-- auto_fixer.py                 # NEW: auto-fix suggestions (~280 lines)
+-- requirements.txt              # MODIFIED: add jira>=3.5.0
+-- env_example.txt               # MODIFIED: add JIRA and AUTOFIX env vars
+-- README.md                     # MODIFIED: add new commands
+-- GUIDE.md                      # MODIFIED: add new feature docs
+-- context.md                    # MODIFIED: update with new architecture
+-- feature_1_output_system.md    # Planning doc (can delete after implementation)
+-- feature_2_jira_integration.md # Planning doc (can delete after implementation)
+-- feature_3_walkthrough.md      # Planning doc (can delete after implementation)
+-- feature_4_autofix.md          # Planning doc (can delete after implementation)
+-- plan.md                       # THIS FILE (can delete after implementation)
```

---

## Verification Plan

### After Phase 0 (Bug Fix)
1. Run `python3 app.py` — should start without errors
2. Send a mock PR webhook payload — should post review as string comment, not crash on `.get('file_comments')`

### After Phase 1 (Output Formatter)
1. Run `python3 -c "from output_formatter import *; print(format_processing_message('Test'))"` — should print formatted message
2. Trigger `\ansieyes_prreview` on test PR — verify output has: header with timestamp, file changes table, clean review content, footer
3. Trigger `\ansieyes_triage` on test issue — verify output has: classification table with type/severity badges, collapsible librarian files, analysis section, footer
4. Check all error/status messages use new format (no more raw inline strings)

### After Phase 2a (JIRA)
1. Set `JIRA_ENABLED=true` with test Atlassian Cloud credentials
2. Trigger `\ansieyes_triage` on test issue
3. Verify: JIRA ticket created with correct summary (`[GitHub #N] title`), correct issue type, correct priority
4. Verify: GitHub comment posted with JIRA ticket link
5. Verify: JIRA key label added to GitHub issue
6. Set `JIRA_ENABLED=false`, trigger triage again — verify no JIRA ticket created, triage works normally

### After Phase 2b (Walkthrough)
1. Trigger `\ansieyes_walkthrough` on test PR with multiple file changes
2. Verify: Mermaid sequence diagram renders properly in GitHub
3. Verify: Files grouped logically by feature/area
4. Verify: Impact analysis lists relevant unmodified files
5. Try `\ansieyes_walkthrough` on an issue — should show error message pointing to `\ansieyes_triage`

### After Phase 3 (Auto-Fix)
1. Create a test PR with a known security issue (e.g., `API_KEY = "hardcoded"`)
2. Trigger `\ansieyes_prreview`
3. Verify: Review comment posted first
4. Verify: Follow-up auto-fix comment posted with diff-style suggestion
5. Verify: Fix has collapsible section, original/suggested code, explanation
6. Set `AUTOFIX_ENABLED=false`, trigger again — verify no fix suggestions
7. Create PR with only minor issues — verify no fix suggestions (threshold works)

### Final Integration Test
1. Run all 3 commands (`\ansieyes_prreview`, `\ansieyes_triage`, `\ansieyes_walkthrough`) on the same repo
2. Verify: No command interferes with another
3. Verify: No crashes, no duplicate comments
4. Verify: All comments have consistent Ansieyes branding

---

## Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| Mermaid diagram syntax errors from Gemini | Broken rendering in GitHub | `_validate_mermaid()` sanitizer; fallback to empty diagram; wrap in collapsible section |
| AI-Issue-Triage CLI lacks `--format json` | Can't get structured PR review data | Passthrough mode: wrap raw markdown in template. Structured parsing is best-effort via regex |
| JIRA API auth failure | No JIRA ticket created | JIRA wrapped in try/except, never blocks triage. Log warnings. Test connection at startup |
| Gemini rate limits during auto-fix (multiple API calls) | Slow or partial fix suggestions | Limit to max 3 fixes. Timeout per fix. Single batched prompt where possible |
| Large PR patches exceeding Gemini context | Truncated analysis or API error | Truncate patches to 80 lines per file, 30 files max |
| Breaking existing behavior during formatter migration | Production regression | Keep original format functions as wrappers that delegate. Never delete, only redirect |
| JIRA project doesn't have expected issue types | JIRA API 400 error | Fall back to "Task" type. Catch and log specific JIRA errors |
| Auto-fix severity parsing is fragile | Misses findings or false triggers | Multiple extraction strategies. Err on under-triggering. Clear "AI-generated" disclaimer |
| GitHub webhook 10s timeout during auto-fix | GitHub marks delivery failed | Existing issue for triage too. Accept timeout, rely on retry. Future: async task queue |
