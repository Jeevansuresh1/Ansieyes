# Feature 4: Auto-Fix Agent

## Problem
PR reviews identify problems but leave fixing to humans. Developers read the review, context-switch back to code, figure out the fix, and commit it. For straightforward issues (missing error handling, hardcoded secrets, obvious bugs), the AI can suggest concrete fixes.

## Solution
After `\ansieyes_prreview` completes, automatically analyze HIGH+ severity findings and generate code fix suggestions posted as a follow-up comment. Phase 1 is suggestion-only (no actual commits).

## Trigger
- Automatic: After every `\ansieyes_prreview` that finds HIGH/CRITICAL issues
- Controlled by: `AUTOFIX_ENABLED` env var (default: true)
- Threshold: `AUTOFIX_MIN_SEVERITY` env var (default: "high")

## Architecture

```
\ansieyes_prreview completes
    |
    v
Review text posted to PR
    |
    v
[NEW] auto_fixer.analyze_for_fixes(review_text, file_changes)
    |
    +-- Extract high-severity findings from review text (regex/heuristics)
    +-- For each finding (max 3):
    |     +-- Send finding + relevant file patch to Gemini
    |     +-- Request structured JSON fix: {original_code, fixed_code, explanation}
    +-- Return structured fix suggestions
    |
    v
output_formatter.format_fix_suggestions(fix_result)
    |
    v
Post as SEPARATE follow-up comment on PR
```

## New File: `auto_fixer.py`

```python
class AutoFixer:
    __init__(self, api_key)
    analyze_for_fixes(review_text, file_changes, repo_url) -> dict
    _extract_high_severity_findings(review_text) -> list
    _generate_fix(finding, file_patch, filename) -> dict
```

## Phase 1 (Current Implementation): Suggestion Mode

Post fix suggestions as comments with diff-style code blocks.

### Output Example
```markdown
# Ansieyes Auto-Fix Suggestions

Found **2** high-severity issues with suggested fixes.

---

<details open>
<summary>:red_circle: Fix 1: Hardcoded API key in auth.py</summary>

**Finding**: API key is hardcoded directly in source code at line 15.
**Severity**: Critical | **File**: `auth.py`

```diff
- API_KEY = "sk-abc123secret"
- headers = {"Authorization": f"Bearer {API_KEY}"}
+ API_KEY = os.getenv("API_KEY")
+ if not API_KEY:
+     raise ValueError("API_KEY environment variable is required")
+ headers = {"Authorization": f"Bearer {API_KEY}"}
```

**Why this fix**: Hardcoded secrets can be exposed through version control.
Moving to environment variables follows the 12-factor app methodology.

</details>

<details>
<summary>:orange_circle: Fix 2: Missing null check in user_handler.py</summary>

**Finding**: `user.email` accessed without null check at line 42.
**Severity**: High | **File**: `user_handler.py`

```diff
- send_notification(user.email, message)
+ if user and user.email:
+     send_notification(user.email, message)
+ else:
+     logger.warning(f"Cannot notify user {user_id}: no email address")
```

**Why this fix**: `user` could be None if the database lookup fails,
causing an AttributeError in production.

</details>

---
> :warning: These are AI-generated suggestions. Review carefully before applying.
> Powered by Ansieyes
```

## Phase 2 (Future): Commit Mode

When the GitHub App gets `Contents: Write` permission:
1. Create branch `ansieyes-fix/<pr-number>`
2. Apply fixes as commits
3. Open a PR targeting the original PR's branch
4. Post link on the original PR
5. Human reviews and merges

This is NOT implemented in Phase 1 - method is stubbed with `NotImplementedError`.

## Severity Extraction Strategy

The review text from AI-Issue-Triage contains patterns like:
- `:red_circle: Critical:` or `**Critical**:` or `### Critical Issues`
- `:orange_circle: High:` or severity keywords
- File references like `auth.py:15` or `**File**: auth.py`

Extraction uses multiple strategies:
1. Regex for known patterns from the output_formatter
2. Keyword search for "critical", "high", "security", "vulnerability"
3. If structured JSON is available from Feature 1's output, use that directly

## Limits
- Max 3 fix suggestions per review (avoid Gemini rate limits)
- Only HIGH and CRITICAL by default (configurable)
- 5-minute total timeout for all fix generation
- Never auto-merge or commit (Phase 1)

## Environment Variables
```
AUTOFIX_ENABLED=true
AUTOFIX_MIN_SEVERITY=high   # critical, high, medium
```

## No New Dependencies
Uses existing `google-generativeai`.

## Safety
- Auto-fix NEVER blocks or delays the PR review. Review is posted first, fixes come as a follow-up.
- If fix generation fails, only logs the error - review is unaffected.
- All fixes wrapped in try/except per-finding (one failure doesn't block others).
- Clear disclaimer: "AI-generated suggestions. Review carefully."
