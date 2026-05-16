# Ansieyes - Complete Project Context

> **Generated**: 2026-05-16
> **Purpose**: Full codebase context reference so any AI assistant or developer can understand the entire project without re-reading every file.

---

## 1. Project Overview

**Ansieyes** is an AI-powered GitHub Bot that performs two core functions:

1. **PR Code Review** (`\ansieyes_prreview`) - Automated pull request reviews using Google's Gemini AI
2. **Issue Triage** (`\ansieyes_triage`) - Two-pass intelligent issue analysis using the external `AI-Issue-Triage` package

The bot is implemented as a **GitHub App** that listens for webhook events (issue comments, pull requests, workflow runs) and responds with AI-generated analysis posted as GitHub comments.

**Key Principle**: Commands are **exact match only** - no extra text is allowed in the trigger comment.

---

## 2. Architecture

```
GitHub Webhook Event (POST /webhook)
        |
        v
   app.py (Flask server, port 3000)
        |
        +-- Signature verification (HMAC SHA-256)
        +-- Authorization check (write access required)
        |
        +-- Event routing:
        |       |
        |       +-- pull_request (opened/synchronize/reopened)
        |       |       --> review_pr() --> pr_reviewer.py --> AI-Issue-Triage CLI (cli.pr_review)
        |       |
        |       +-- issue_comment (created)
        |       |       +-- "\ansieyes_prreview" on PR --> handle_pr_review_mention() --> pr_reviewer.py
        |       |       +-- "\ansieyes_triage" on Issue --> handle_triage_mention() --> issue_triager.py
        |       |       +-- Wrong context --> Helpful error message
        |       |
        |       +-- workflow_run (completed)
        |               --> analyze_workflow_run() --> pr_reviewer.py (workflow analysis)
        |
        +-- Results posted as GitHub comments
        +-- Labels auto-applied (triage only)
```

### Two-Pass Triage Architecture (issue_triager.py)

```
Step 0: Prompt Injection Check (local, using AI-Issue-Triage detector)
   |-- HIGH/CRITICAL risk --> BLOCK, post security alert
   |-- LOW/MEDIUM risk --> LOG and continue
   |
Step 1: Duplicate Detection (AI-Issue-Triage CLI: cli.duplicate_check)
   |-- Only checks OPEN issues created BEFORE the current issue
   |-- Duplicate found --> Post duplicate report, skip analysis
   |
Step 2: Librarian Pass (AI-Issue-Triage CLI: cli.librarian)
   |-- Generates repomix chunks per directory (respects .omit-triage)
   |-- Identifies relevant files from codebase
   |
Step 3: Surgeon Pass (AI-Issue-Triage CLI: cli.analyze)
   |-- Deep analysis with targeted repomix of identified files
   |-- Respects triage.config.json settings (custom model, custom prompt)
   |-- Outputs type classification, severity, root cause, proposed solutions
```

---

## 3. File-by-File Breakdown

### 3.1 Core Application Files

#### `app.py` (962 lines) - Main Entry Point

- **Flask web server** with two routes: `/health` (GET) and `/webhook` (POST)
- **Webhook signature verification**: HMAC SHA-256 using `GITHUB_WEBHOOK_SECRET`
- **GitHub App authentication**: Supports two methods for private key:
  - `GITHUB_PRIVATE_KEY_B64` (base64-encoded env var, preferred for cloud)
  - `GITHUB_PRIVATE_KEY_PATH` (file path fallback)
- **Authorization gate** (`is_authorized_user()`): Only users with `admin`, `maintain`, or `write` permission can trigger commands. Unauthorized users get a denial comment.
- **Bot loop prevention**: Comments from accounts ending in `[bot]` are ignored.
- **Context validation**: `\ansieyes_triage` only works on issues (not PRs), `\ansieyes_prreview` only on PRs (not issues). Wrong usage posts a helpful error.
- **Label management** (triage only):
  - Removes ALL existing labels on the issue first
  - Applies new labels based on triage results:
    - Type labels: `Type : Bug`, `Type : Enhancement`, `Type : Feature Request`
    - Severity labels: `Severity : Critical/High/Medium/Low`
    - Status labels: `ai-triaged`, `duplicate`, `Prompt injection blocked`
  - Auto-creates labels in repo if they don't exist (with color coding via `get_label_color()`)
- **Label color scheme** (`get_label_color()`):
  - Bugs = red (`d73a4a`), Enhancements = light blue (`a2eeef`), Features = green (`0e8a16`)
  - Severity critical = dark red, high = orange-red, medium = yellow, low = green
  - AI-processed = purple (`7057ff`), Security = dark red (`b60205`)
- **Workflow run analysis**: Finds associated PR by matching `head_sha`, collects job/step details, generates AI analysis
- **Temporary repo cloning**: Clones target repo (shallow, `--depth 1`) to read config files, with 5-minute timeout and cleanup

#### `pr_reviewer.py` (131 lines) - PR Review Engine

- **Class `PRReviewer`**: Delegates to `AI-Issue-Triage` CLI tool (`cli.pr_review`)
- **`review_pr()`**:
  - Writes PR data (title, body, file changes with patches) to a temp JSON file
  - Runs `python3 -m cli.pr_review --pr-file <path> --output <path> --format markdown` in the AI-Issue-Triage directory
  - 5-minute subprocess timeout
  - Replaces "AI Code Review" header with "Ansieyes Report" in output
  - Returns formatted markdown string (not a dict)
- **`format_review_summary()`**: Passthrough - AI-Issue-Triage already formats output
- **Dependencies**: Requires `AI_TRIAGE_PATH` to point to a valid AI-Issue-Triage installation

#### `issue_triager.py` (723 lines) - Issue Triage Engine

- **Class `IssueTriager`**: Implements the full two-pass triage pipeline
- **`__init__()`**:
  - Adds AI-Issue-Triage to `sys.path` for importing `utils.security.prompt_injection`
  - Gracefully handles import failures (detection becomes disabled)
- **`check_prompt_injection()`**:
  - Uses AI-Issue-Triage's `detect_prompt_injection()` with `strict_mode=False`
  - Returns risk level (safe/low/medium/high/critical), confidence, detected patterns
  - Falls back to safe if detector unavailable
- **`check_for_duplicates()`**:
  - Runs `python3 -m cli.duplicate_check` via subprocess
  - Inputs: title, description, existing issues JSON file
  - Returns JSON with `is_duplicate`, `duplicate_of`, `similarity_score`, etc.
- **`run_librarian()`** (Pass 1):
  - Generates repomix chunks if not provided
  - Runs `python3 -m cli.librarian` with chunks directory
  - Returns list of `relevant_files`
- **`run_surgeon()`** (Pass 2):
  - Runs `python3 -m cli.analyze` with targeted repomix file
  - Supports custom Gemini model and custom prompt from `triage.config.json`
  - Returns `formatted_output` (text format)
- **`_generate_repomix_chunks()`**:
  - Reads `.omit-triage` file for exclusions
  - Iterates root directories, runs `repomix` for each (with `--compress`, `--remove-comments`, `--remove-empty-lines`)
  - Finds repomix via PATH, npx, or common node_modules paths
- **`_load_triage_config()`**: Reads `triage.config.json` from repo root
- **`triage_issue()`** (orchestrator):
  - Full pipeline: injection check -> duplicate check -> librarian -> targeted repomix -> surgeon
  - Handles its own temp directory if no `repo_path` provided
  - Returns dict with keys: `prompt_injection_check`, `duplicate_check`, `librarian`, `surgeon`
- **`format_triage_comment()`**:
  - Formats results as GitHub-flavored markdown
  - Three output formats: security alert (blocked), duplicate report, or full analysis report
  - Replaces "Gemini Analysis Report" branding with "Ansieyes Report"
  - Extracts type/severity from formatted output using regex patterns:
    - Type: `\*\*Type:\*\*\s+\`([^`]+)\``
    - Severity: `\*\*Severity:\*\*\s+\`([^`]+)\``

#### `test_bot.py` (90 lines) - Manual Test Script

- Tests Gemini API connectivity and basic PR review generation
- Uses mock PR data (hardcoded auth.py example)
- **Note**: References `reviewer.model.generate_content()` which doesn't exist in current `PRReviewer` class (legacy code from when PRReviewer used Gemini directly)
- Run with: `python3 test_bot.py`

### 3.2 Configuration Files

#### `prompt_config.yml` - Repository-Specific Prompt Templates

- Maps repository URL patterns (regex) to prompt types: `network`, `devtools`, `default`
- **Network repos** (`ansible-collections/ansible.*`, `ansible-collections/cisco.*`, `ansible-collections/arista.*`):
  - Focus: Ansible playbooks, network device configs, security, idempotency
- **DevTools repos** (`ansible/.*`):
  - Focus: API design, CLI usability, documentation, developer experience
- **Default**: Generic code review
- Each type has: `system_role`, `review_structure`, `workflow_analysis` templates
- **Note**: This config is defined but NOT currently loaded by `pr_reviewer.py` (the current PRReviewer delegates to AI-Issue-Triage CLI, which handles its own prompts)

#### `triage.config.example.json` - Per-Repository Triage Config

- Placed in target repository roots (not in Ansieyes itself)
- Controls: duplicate detection toggle, security check toggle, auto-labeling
- Librarian/Surgeon model selection (default: `gemini-2.0-flash-001`)
- Custom trigger commands, label prefixes, omit directories
- Custom prompt path support (resolved relative to repo root)

#### `requirements.txt` - Python Dependencies

- **Flask 3.0.0** - Web framework
- **requests 2.31.0** - HTTP client
- **google-generativeai 0.3.1** - Gemini API (legacy direct usage)
- **python-dotenv 1.0.0** - .env file loading
- **PyGithub 2.1.1** - GitHub API client
- **cryptography 41.0.7** - GitHub App JWT authentication
- **PyYAML 6.0.1** - YAML config parsing
- **google-genai** - Newer Gemini client (for AI-Issue-Triage)
- **pydantic >= 2.0.0** - Data validation
- **scikit-learn >= 1.3.0** - ML utilities (for AI-Issue-Triage)
- **numpy >= 1.24.0** - Numerical computing
- **pytector >= 0.1.0** - Prompt injection detection

#### `env_example.txt` - Environment Variable Template

Required vars:
- `GEMINI_API_KEY` - Google Gemini API key
- `GITHUB_APP_ID` - GitHub App numeric ID
- `GITHUB_PRIVATE_KEY_PATH` - Path to .pem file
- `GITHUB_WEBHOOK_SECRET` - Webhook HMAC secret
- `AI_TRIAGE_PATH` - Path to AI-Issue-Triage clone (default: `/Users/shvenkat/Documents/AI/AI-Issue-Triage`)
- `PORT` (default: 3000), `HOST` (default: 0.0.0.0)

Optional:
- `GITHUB_PRIVATE_KEY_B64` - Base64-encoded private key (alternative to file path)
- `PROMPT_CONFIG_PATH` - Custom prompt config location

#### `.gitignore`

Ignores: `.env`, `__pycache__/`, `*.pyc`, `venv/`, `*.pem`, `*.key`, `.DS_Store`, `*.log`

### 3.3 Deployment Files

#### `Dockerfile`

- Base: `python:3.11-slim`
- Copies requirements, installs deps, copies app code
- Exposes port 3000, runs `python app.py`
- **Note**: Does NOT install Node.js/repomix (needed for triage), so Docker deployment only supports PR review out of the box

#### `docker-compose.yml`

- Single service `Ansieyes`, builds from Dockerfile
- Maps port 3000, reads `.env` file
- Mounts `private-key.pem` as read-only volume
- Restart policy: `unless-stopped`

#### `Procfile`

- `web: python app.py` (for Heroku/Railway)

#### `ecosystem.config.js`

- PM2 process manager config
- Runs `app.py` with `python3` interpreter
- Working dir: `/home/ubuntu/Ansieyes`
- Log files: `/home/ubuntu/logs/`
- Auto-restart enabled, 1GB memory limit, single instance

#### `render.yaml`

- Render.com service definition
- Python environment, health check at `/health`

#### `railway.json`

- Railway deployment config using Nixpacks builder
- Restart on failure with max 10 retries

#### `aws-task-definition.json`

- AWS ECS Fargate task definition
- 256 CPU units, 512 MB memory
- Container port 3000, CloudWatch logging
- Health check: `curl -f http://localhost:3000/health`

#### `aws-deploy.sh` (129 lines)

- EC2 deployment script: installs system deps, clones repo, creates venv
- Sets up Nginx reverse proxy, installs PM2, starts application
- Must NOT run as root

### 3.4 Setup Scripts

#### `setup-ansieyes.sh` (1184 lines) - Comprehensive Setup Script

- Interactive menu with 5 options:
  1. Complete Setup (prerequisites, Node.js, repomix, AI-Issue-Triage, dependencies, config, test)
  2. Quick Setup (config + test only)
  3. EC2 Production Deployment (systemd service, nginx + SSL)
  4. Test Existing Setup
  5. Exit
- Handles multiple OS: macOS (Homebrew), Debian/Ubuntu (apt), RHEL/CentOS (yum)
- Auto-detects EC2 environment via instance metadata
- Handles externally-managed Python environments (`--break-system-packages`)
- Sets up systemd service with proper user/path/env configuration
- Configures Nginx with Let's Encrypt SSL via certbot
- Adds `~/.local/bin` to PATH permanently
- Node.js 20+ requirement (for repomix compatibility)
- Clones AI-Issue-Triage from `https://github.com/shvenkat-rh/AI-Issue-Triage.git`, checks out `feature/pr-analyzer` branch

#### `setup.sh` (47 lines) - Simple Setup Script

- Creates Python venv, installs requirements, copies `.env.example` to `.env`
- Legacy/minimal alternative to `setup-ansieyes.sh`

#### `scripts/encode_key.sh`

- Utility to base64-encode a GitHub App private key (.pem file)
- Output format: `GITHUB_PRIVATE_KEY_B64=<encoded>`

### 3.5 Documentation Files (`docs/`)

| File | Purpose |
|------|---------|
| `GUIDE.md` | **Primary documentation** - Complete setup, usage, architecture, FAQ, troubleshooting |
| `README.md` | Brief overview, points to GUIDE.md |
| `QUICKSTART.md` | Minimal steps: get API key, create GitHub App, configure, run |
| `LOCAL_TESTING.md` | Four testing methods: API test, Flask test, Docker test, full webhook with ngrok |
| `HOSTING.md` | 7 hosting options compared: Railway, Render, Heroku, EC2, Docker, GitHub Actions, DigitalOcean |
| `DEPLOYMENT_QUICKSTART.md` | Fastest deploy paths (Railway 5min, Render, Docker) |
| `AWS_DEPLOYMENT.md` | Detailed AWS guide: EC2, ECS/Fargate, Elastic Beanstalk, cost estimation, security |
| `AWS_QUICKSTART.md` | Condensed EC2 deployment steps |
| `PROMPT_CONFIGURATION.md` | How prompt_config.yml works (repo URL pattern matching, per-type prompts) |

---

## 4. External Dependencies

### AI-Issue-Triage (Critical External Dependency)

- **Repository**: `https://github.com/shvenkat-rh/AI-Issue-Triage`
- **Branch**: `feature/pr-analyzer`
- **Must be cloned locally** and path set via `AI_TRIAGE_PATH` env var
- **Used CLI modules**:
  - `cli.pr_review` - PR review analysis
  - `cli.duplicate_check` - Duplicate issue detection
  - `cli.librarian` - File identification (Pass 1)
  - `cli.analyze` - Deep analysis (Pass 2)
- **Used Python imports**:
  - `utils.security.prompt_injection.detect_prompt_injection` - Injection detection
  - `utils.security.prompt_injection.InjectionRisk` - Risk level enum

### Repomix (Required for Issue Triage)

- **npm package**: `repomix`
- **Purpose**: Generates compressed text representations of repository directories
- **Requires**: Node.js 20+
- **Usage**: Called as subprocess by `issue_triager.py`

---

## 5. Event Flow Details

### PR Review Flow (`\ansieyes_prreview`)

1. User comments `\ansieyes_prreview` on a PR
2. Bot verifies webhook signature
3. Bot checks comment is exact match, not from a bot, and user has write access
4. Bot validates this is a PR (not an issue)
5. Posts "Ansieyes PR Review Initiated" processing comment
6. Fetches PR file changes via GitHub API (filename, status, additions, deletions, patch)
7. Calls `pr_reviewer.review_pr()` which delegates to AI-Issue-Triage CLI
8. Posts review as issue comment, deletes processing comment
9. **No labels are applied** for PR reviews

### Issue Triage Flow (`\ansieyes_triage`)

1. User comments `\ansieyes_triage` on an issue
2. Same auth checks as above
3. Bot validates this is an issue (not a PR)
4. Posts "Ansieyes Issue Triage has been Initiated" processing comment
5. Clones the repository (`git clone --depth 1`, 5-min timeout) to get config files
6. Fetches all open issues created before this issue (for duplicate detection)
7. Calls `issue_triager.triage_issue()`:
   - Checks prompt injection (blocks on HIGH/CRITICAL)
   - Checks duplicates against older open issues
   - Runs Librarian (identifies relevant files)
   - Generates targeted repomix with identified files
   - Runs Surgeon (deep analysis with type/severity/root cause)
8. Posts formatted triage comment
9. Deletes processing comment
10. **Label management**:
    - Removes ALL existing labels
    - If prompt injection blocked: adds `Prompt injection blocked`
    - If duplicate: adds `duplicate` + `ai-triaged`
    - If normal: adds `Type : <type>`, `Severity : <severity>`, `ai-triaged`
    - Auto-creates labels with appropriate colors if they don't exist

### Auto PR Review Flow (on PR events)

1. PR is opened, synchronized, or reopened
2. Same pipeline as manual `\ansieyes_prreview` but triggered automatically
3. Posts review as issue comment + inline review comments on specific files/lines

### Workflow Run Analysis Flow

1. GitHub Actions workflow completes
2. Bot finds the associated PR (by branch + head SHA, checks open then closed PRs)
3. Collects job names, conclusions, step details
4. Calls `pr_reviewer.analyze_workflow_run()` for AI analysis
5. Posts formatted comment with status emoji, failed jobs list, and analysis

---

## 6. Security Model

- **Webhook signature verification**: HMAC SHA-256 (skipped if secret not configured, with warning)
- **Authorization**: Only repository collaborators with `admin`, `maintain`, or `write` permission can trigger commands
- **Prompt injection detection**: Uses AI-Issue-Triage's pattern-based detector
  - LOW/MEDIUM risk: logged but analysis continues
  - HIGH/CRITICAL risk: analysis blocked, security alert posted
- **Bot loop prevention**: Ignores comments from `*[bot]` accounts
- **Temporary file cleanup**: All cloned repos and temp files are cleaned up in finally blocks
- **Private key handling**: Supports base64 env var (no file on disk) for cloud deployments

---

## 7. Configuration Hierarchy

```
1. Environment variables (.env file)
   |-- GEMINI_API_KEY, GITHUB_APP_ID, etc.
   |
2. prompt_config.yml (in Ansieyes repo)
   |-- Repository URL -> prompt type mapping
   |-- Per-type system roles and review structures
   |-- NOTE: Currently NOT actively loaded by the codebase
   |
3. Per-repository configs (in TARGET repos, fetched at triage time)
   |-- triage.config.json -> Custom Gemini model, custom prompt path
   |-- .omit-triage -> Directories to exclude from repomix
```

---

## 8. Known Considerations

1. **`prompt_config.yml` is defined but not loaded**: The current `PRReviewer` delegates everything to AI-Issue-Triage CLI, which has its own prompt system. The YAML config was likely from an earlier version when `PRReviewer` called Gemini directly.

2. **`test_bot.py` references `reviewer.model`**: The test script calls `reviewer.model.generate_content()` but `PRReviewer` no longer has a `model` attribute (it uses subprocess calls to AI-Issue-Triage). This test would fail at that line.

3. **Docker image lacks Node.js/repomix**: The Dockerfile only installs Python deps. Issue triage (`\ansieyes_triage`) requires Node.js and repomix, which aren't installed in the container. PR review would work, but triage would fail.

4. **Hardcoded default path**: `AI_TRIAGE_PATH` defaults to `/Users/shvenkat/Documents/AI/AI-Issue-Triage` (a developer's local path). Production deployments must override this.

5. **Label removal is aggressive**: `handle_triage_mention()` removes ALL existing labels before applying new ones. This could remove manually-applied labels unrelated to triage.

6. **Synchronous processing**: Webhook handlers process requests synchronously. Long triage operations (clone + 4 AI calls) could cause GitHub webhook timeouts (~10s). The comment "Process PR review asynchronously" in code is aspirational - it's actually synchronous.

7. **`review_pr()` in `app.py` uses old dict-based return**: The `review_pr()` function in `app.py` (line 351) calls `pr_reviewer.review_pr()` and expects a dict with `file_comments`, but the current `PRReviewer.review_pr()` returns a string. The auto-trigger flow (PR opened/synchronized) may have issues posting inline comments.

8. **Dual Gemini packages**: `requirements.txt` includes both `google-generativeai==0.3.1` (legacy) and `google-genai` (newer, for AI-Issue-Triage). Both are imported in `app.py`.

---

## 9. Git History Summary

The project has evolved through these phases:
- Initial PR review bot with direct Gemini API integration
- Addition of issue triage with AI-Issue-Triage two-pass architecture
- Label management system (multiple iterations fixing label logic)
- Duplicate detection with chronological ordering
- Prompt injection security layer
- Trigger command changes and exact-match enforcement
- Authorization/permission checks for command execution (most recent)
- Setup script consolidation into `setup-ansieyes.sh`

---

## 10. Repository Structure

```
Ansieyes/
+-- app.py                          # Main Flask application (webhook handler)
+-- pr_reviewer.py                  # PR review engine (delegates to AI-Issue-Triage CLI)
+-- issue_triager.py                # Issue triage engine (two-pass architecture)
+-- test_bot.py                     # Manual test script
+-- prompt_config.yml               # Repository-type prompt templates (not actively loaded)
+-- requirements.txt                # Python dependencies
+-- env_example.txt                 # Environment variable template
+-- triage.config.example.json      # Per-repo triage config example
+-- .gitignore                      # Git ignore rules
+-- Dockerfile                      # Docker container definition
+-- docker-compose.yml              # Docker Compose service definition
+-- Procfile                        # Heroku/Railway process definition
+-- ecosystem.config.js             # PM2 process manager config
+-- render.yaml                     # Render.com deployment config
+-- railway.json                    # Railway deployment config
+-- aws-task-definition.json        # AWS ECS Fargate task definition
+-- aws-deploy.sh                   # AWS EC2 deployment script
+-- setup.sh                        # Simple setup script (legacy)
+-- setup-ansieyes.sh               # Comprehensive interactive setup script
+-- README.md                       # Brief project overview
+-- GUIDE.md                        # Complete setup & usage guide
+-- scripts/
|   +-- encode_key.sh               # Base64 encode GitHub private key
+-- docs/
    +-- QUICKSTART.md               # Minimal setup steps
    +-- LOCAL_TESTING.md            # Local testing methods
    +-- HOSTING.md                  # Hosting platform comparison
    +-- DEPLOYMENT_QUICKSTART.md    # Quick deploy guide
    +-- AWS_DEPLOYMENT.md           # Detailed AWS deployment
    +-- AWS_QUICKSTART.md           # AWS quick reference
    +-- PROMPT_CONFIGURATION.md     # Prompt config documentation
```

---

## 11. Tech Stack Summary

| Layer | Technology |
|-------|-----------|
| Language | Python 3.8+ |
| Web Framework | Flask 3.0.0 |
| AI Provider | Google Gemini (via AI-Issue-Triage) |
| GitHub Integration | PyGithub 2.1.1 + GitHub App (JWT auth) |
| Code Indexing | Repomix (Node.js) |
| Security | HMAC webhook verification, prompt injection detection, permission checks |
| Process Management | PM2 / systemd |
| Reverse Proxy | Nginx (with Let's Encrypt SSL) |
| Containerization | Docker |
| Deployment Targets | AWS EC2, ECS/Fargate, Railway, Render, Heroku, DigitalOcean |
