# ADM Silver Layer — Complete Developer Setup & CI/CD Security Automation Guide
**Audience:** Every developer joining the ADM team + DevOps setting up GitLab pipelines  
**Repos:** `apps-azure` · `apis-azure` · `adm-foundation-layer-agents-azure`  
**Stack:** Python (FastAPI, Azure Functions) + React/TypeScript  
**Platform:** GitLab (self-hosted, closed-source)

---

## How This Works — Big Picture First

Before touching anything, understand the 3 trigger points:

```
┌─────────────────────────────────────────────────────────────────────┐
│  TRIGGER 1: git commit (local)                                      │
│  → pre-commit hooks fire BEFORE the commit is accepted              │
│  → Catches: secrets, lint errors, format issues, verify=False       │
│  → If any hook fails: commit is REJECTED, nothing goes to git       │
├─────────────────────────────────────────────────────────────────────┤
│  TRIGGER 2: git push (local → GitLab)                               │
│  → pre-push hooks fire BEFORE push reaches GitLab                  │
│  → Catches: mypy type errors, heavier checks you skip on commit     │
│  → If hook fails: push is REJECTED locally                          │
├─────────────────────────────────────────────────────────────────────┤
│  TRIGGER 3: Merge Request opened on GitLab                          │
│  → .gitlab-ci.yml pipeline runs on GitLab server                   │
│  → Catches: SAST (bandit, semgrep), full dependency audit,          │
│             SonarQube analysis, coverage gate                        │
│  → If pipeline fails: MR is BLOCKED, cannot be merged              │
└─────────────────────────────────────────────────────────────────────┘
```

The philosophy: **catch problems as early and cheaply as possible.**  
A secret caught locally = zero risk. A secret caught in GitLab = already in git history = must rotate credentials.

---

## PART 1 — DEVELOPER LOCAL SETUP

---

### STEP 1 — Prerequisites: What every developer needs installed

Every developer on the team needs these tools on their machine before anything else.

#### 1.1 Check what you already have

Open a terminal and run:

```bash
python --version      # Need 3.10 or higher
node --version        # Need 18 or higher (for apps-azure devs)
git --version         # Need 2.x
pip --version
```

**Expected output:**
```
Python 3.11.9
v20.11.0
git version 2.44.0
pip 24.0 from ...
```

If Python < 3.10, install from python.org. If Node < 18, use nvm to upgrade.

#### 1.2 Install pre-commit

```bash
pip install pre-commit
```

**Expected output:**
```
Collecting pre-commit
  Downloading pre_commit-3.7.1-py2.py3-none-any.whl (218 kB)
Successfully installed pre-commit-3.7.1
```

Verify:
```bash
pre-commit --version
```
```
pre-commit 3.7.1
```

#### 1.3 Install gitleaks (secret scanner — standalone binary)

**macOS:**
```bash
brew install gitleaks
```

**Linux (Ubuntu/Debian):**
```bash
# Download the binary directly
curl -sSfL https://github.com/gitleaks/gitleaks/releases/download/v8.18.4/gitleaks_8.18.4_linux_x64.tar.gz \
  | tar -xz -C /usr/local/bin gitleaks
chmod +x /usr/local/bin/gitleaks
```

**Windows (Git Bash / WSL recommended):**
```bash
# Download gitleaks_8.18.4_windows_x64.zip from:
# https://github.com/gitleaks/gitleaks/releases
# Extract and add to PATH
```

Verify:
```bash
gitleaks version
```
```
v8.18.4
```

#### 1.4 Install Python dev tools

Each Python repo has a `requirements-dev.txt`. Install these in your virtualenv:

```bash
cd apis-azure   # or agents-azure
python -m venv .venv
source .venv/bin/activate          # Linux/macOS
# OR: .venv\Scripts\activate       # Windows

pip install -r requirements.txt
pip install -r requirements-dev.txt
```

**Contents of `requirements-dev.txt` (you create this once, lives in repo):**
```
pre-commit==3.7.1
ruff==0.4.7
black==24.4.2
mypy==1.10.0
bandit[toml]==1.7.9
pytest==8.2.2
pytest-cov==5.0.0
pytest-asyncio==0.23.7
httpx==0.27.0
pip-audit==2.7.3
types-requests
types-PyYAML
```

#### 1.5 Install Node/frontend tools (apps-azure devs only)

```bash
cd apps-azure
npm install
```

This picks up `eslint`, `prettier`, `typescript` from `package.json`. Verify:

```bash
npx eslint --version
npx prettier --version
npx tsc --version
```
```
v8.57.0
3.3.2
Version 5.4.5
```

---

### STEP 2 — Clone the repo and activate pre-commit

This is what every developer does **once** per repo after cloning.

#### 2.1 Clone

```bash
git clone https://gitlab.yourtiger.com/adm/apis-azure.git
cd apis-azure
```

#### 2.2 Create and activate virtualenv

```bash
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt -r requirements-dev.txt
```

#### 2.3 Activate pre-commit hooks

```bash
pre-commit install
pre-commit install --hook-type pre-push
```

**Expected output:**
```
pre-commit installed at .git/hooks/pre-commit
pre-commit installed at .git/hooks/pre-push
```

What this does: it writes two small shell scripts into your `.git/hooks/` folder:
- `.git/hooks/pre-commit` — runs before every `git commit`
- `.git/hooks/pre-push` — runs before every `git push`

These are local to your machine. They don't get committed. Every developer must run this.

#### 2.4 Verify the hooks are installed

```bash
ls -la .git/hooks/
```

**Expected output (relevant lines):**
```
-rwxr-xr-x  pre-commit
-rwxr-xr-x  pre-push
```

#### 2.5 Run pre-commit once against all files (first-time validation)

```bash
pre-commit run --all-files
```

This runs every hook against every file in the repo. On a fresh repo it should pass. On an existing codebase with issues, you'll see failures — that's intentional and expected.

**Expected output (clean codebase):**
```
🔑 Secret scan (gitleaks)....................................Passed
trailing whitespace..........................................Passed
end of file fixer............................................Passed
check yaml...................................................Passed
check json...................................................Passed
detect private key...........................................Passed
🎨 Format check (black)......................................Passed
🔍 Lint (ruff)...............................................Passed
🧠 Type check (mypy).........................................Passed
🚫 Block verify=False........................................Passed
🚫 Block f-string SQL........................................Passed
🚫 Block hardcoded password patterns.........................Passed
🚫 Block committing real .env files..........................Passed
```

**Expected output (codebase with issues):**
```
🔑 Secret scan (gitleaks)....................................Failed
- hook id: gitleaks
- exit code: 1

    Finding:     Rule:    generic-api-key
    File:        backend/config.py
    Line:        12
    Match:       DB_PASSWORD = "tiger@2024!"

🎨 Format check (black)......................................Failed
- hook id: black
- exit code: 1
    would reformat backend/api/routes/agents.py
    Oh no! 💥 💔 💥
    1 file would be reformatted.

🔍 Lint (ruff)...............................................Failed
- hook id: ruff
- exit code: 1
    backend/api/routes/agents.py:45:5: S608 SQL injection via string-based query
    backend/core/db.py:12:1: E402 Module level import not at top of file
```

Fix these before committing. See Step 3 for how to fix each type.

---

### STEP 3 — What developers do when something fails (fixing issues)

This is the most important section for developers day-to-day.

---

#### 3A — Fixing format errors (Black)

**Failure message you'll see:**
```
🎨 Format check (black)......................................Failed
    would reformat backend/api/routes/agents.py
```

**How to fix:**
```bash
black .
```

**Expected output:**
```
reformatted backend/api/routes/agents.py
All done! ✨ 🍰 ✨
1 file reformatted, 12 files left unchanged.
```

Now run `git add .` and `git commit` again. Black will pass this time.

---

#### 3B — Fixing lint errors (Ruff)

**Failure message you'll see:**
```
🔍 Lint (ruff)...............................................Failed
    backend/api/routes/agents.py:45:5: S608 SQL injection via string-based query
    backend/core/db.py:12:1: F401 'os' imported but unused
```

**How to fix — auto-fixable issues:**
```bash
ruff check --fix .
```

**Expected output:**
```
Found 3 errors (3 fixed, 0 remaining).
```

**How to fix — manual issues (SQL injection):**

The `S608` flag means ruff found something like:
```python
# BAD — ruff flagged this
query = f"SELECT * FROM documents WHERE id = {doc_id}"
cursor.execute(query)
```

You must manually change it to:
```python
# GOOD — parameterized query
query = "SELECT * FROM documents WHERE id = %s"
cursor.execute(query, (doc_id,))
```

---

#### 3C — Fixing type errors (mypy)

**Failure message you'll see (on git push):**
```
🧠 Type check (mypy).........................................Failed
    backend/api/routes/agents.py:67: error: Argument 1 to "run_agent"
    has incompatible type "Optional[str]"; expected "str"  [arg-type]
    backend/core/db.py:34: error: Item "None" of "Optional[Connection]"
    has no attribute "cursor"  [union-attr]
    Found 2 errors in 2 files (checked 18 source files)
```

**How to fix:**
```python
# BAD
def run_agent(agent_name: Optional[str]) -> dict:
    return agents[agent_name].run()   # agent_name could be None

# GOOD
def run_agent(agent_name: str) -> dict:
    return agents[agent_name].run()

# OR if None is genuinely possible:
def run_agent(agent_name: Optional[str]) -> dict:
    if agent_name is None:
        raise ValueError("agent_name is required")
    return agents[agent_name].run()
```

---

#### 3D — Fixing secret scan failures (gitleaks)

**Failure message you'll see:**
```
🔑 Secret scan (gitleaks)....................................Failed
    Finding:
    RuleID:      generic-api-key
    Description: Generic API Key
    File:        backend/config.py
    Line:        8
    Match:       DB_PASSWORD = "Tiger@ADM2024!"

    RuleID:      azure-storage-account-key
    File:        agents/document_agent/connector.py
    Line:        23
    Match:       AZURE_STORAGE_KEY = "abc123xyz..."
```

**How to fix:**

1. Remove the secret from the file immediately:

```python
# BAD — what's currently in the file
DB_PASSWORD = "Tiger@ADM2024!"

# GOOD — replace with env var
import os
DB_PASSWORD = os.getenv("DB_PASSWORD")
```

2. Add to `.env` file (which is gitignored):
```
DB_PASSWORD=Tiger@ADM2024!
```

3. Make sure `.env` is in `.gitignore`:
```bash
echo ".env" >> .gitignore
```

4. Create/update `.env.example` with a placeholder (this IS committed):
```
DB_PASSWORD=your-db-password-here
```

5. **CRITICAL — If the secret was already committed in a previous commit:**  
The secret is now in git history. Even after removing it from the file, gitleaks will find it in history.

```bash
# Check if it's in history
git log --all -p | grep "Tiger@ADM2024"

# If yes: you MUST rotate the credential (change the actual password in the DB/Azure)
# Then clean history using git filter-repo (NOT git filter-branch):
pip install git-filter-repo
git filter-repo --path backend/config.py --invert-paths   # nuclear option
# OR use interactive rebase to edit the specific commit
```

**Always rotate the actual secret.** Cleaning history is secondary.

---

#### 3E — Fixing verify=False

**Failure message you'll see:**
```
🚫 Block verify=False........................................Failed
    backend/services/agent_caller.py:34: verify=False
```

**The file contains:**
```python
response = requests.post(agent_url, json=payload, verify=False)  # BAD
```

**How to fix:**
```python
import os

# For internal Azure services (self-signed certs common in dev)
CA_BUNDLE = os.getenv("AZURE_CA_BUNDLE", True)  # True = default system certs
response = requests.post(agent_url, json=payload, verify=CA_BUNDLE)
```

For Azure Functions with managed certs, `verify=True` (the default) works. If you have a custom CA:
```python
# Point to your org's CA bundle
response = requests.post(agent_url, json=payload, verify="/path/to/tiger-ca-bundle.crt")
```

---

#### 3F — Fixing f-string SQL

**Failure message you'll see:**
```
🚫 Block f-string SQL........................................Failed
    backend/api/routes/data.py:89: f"SELECT * FROM {table_name}
```

**How to fix:**

```python
# BAD
query = f"SELECT * FROM {table_name} WHERE user_id = {user_id}"
result = db.execute(query)

# GOOD — parameterized (PostgreSQL with psycopg2)
query = "SELECT * FROM documents WHERE user_id = %s"
result = db.execute(query, (user_id,))

# GOOD — SQLAlchemy ORM (preferred for this project)
from sqlalchemy import text
result = db.execute(
    text("SELECT * FROM documents WHERE user_id = :uid"),
    {"uid": user_id}
)
```

Note: Table names **cannot** be parameterized in SQL. If you genuinely need dynamic table names, use a whitelist:

```python
ALLOWED_TABLES = {"documents", "metadata", "relationships"}
if table_name not in ALLOWED_TABLES:
    raise ValueError(f"Invalid table: {table_name}")
query = f"SELECT * FROM {table_name} WHERE user_id = %s"  # now safe
```

---

#### 3G — Fixing localStorage violations (apps-azure devs)

**Failure message you'll see:**
```
🚫 Block localStorage usage..................................Failed
    src/store/workflowStore.ts:45: localStorage.setItem('authToken', token)
    src/components/DocViewer.tsx:112: localStorage.setItem('workflowState', ...)
```

**How to fix:**

```typescript
// BAD — storing auth token in localStorage
localStorage.setItem('authToken', token);

// GOOD — use MSAL's built-in token cache (already in your stack)
// MSAL handles token storage securely. Don't touch it manually.
import { useMsal } from "@azure/msal-react";
const { instance } = useMsal();
// MSAL stores tokens in sessionStorage with encryption by default
```

```typescript
// BAD — storing workflow state in localStorage
localStorage.setItem('workflowState', JSON.stringify(state));

// GOOD — use React state / Zustand / Redux (in-memory)
import { create } from 'zustand';
const useWorkflowStore = create((set) => ({
  workflowState: null,
  setWorkflowState: (state) => set({ workflowState: state }),
}));
// State lives in memory, not browser storage
// For persistence, save to backend API, not browser
```

---

### STEP 4 — The git commit flow (what developers experience daily)

#### Scenario 1 — Clean commit (everything passes)

```bash
git add backend/api/routes/agents.py
git commit -m "feat: add document classification agent endpoint"
```

**What you see in terminal:**
```
[pre-commit hooks running...]

🔑 Secret scan (gitleaks)....................................Passed
trailing whitespace..........................................Passed
end of file fixer............................................Passed
check yaml...................................................Passed
detect private key...........................................Passed
🎨 Format check (black)......................................Passed
🔍 Lint (ruff)...............................................Passed
🚫 Block verify=False........................................Passed
🚫 Block f-string SQL........................................Passed
🚫 Block hardcoded password patterns.........................Passed
🚫 Block committing real .env files..........................Passed

[main 3f8a2c1] feat: add document classification agent endpoint
 1 file changed, 47 insertions(+)
```

Commit went through. Fast — typically 3–8 seconds for Python files.

---

#### Scenario 2 — Commit blocked by secret

```bash
git add backend/config.py
git commit -m "fix: update db config"
```

**What you see:**
```
🔑 Secret scan (gitleaks)....................................Failed
- hook id: gitleaks
- exit code: 1

        ○
        │╲
        │ ○
        ○ ░
        ░    leak

Finding:     Rule:    generic-api-key
Description: Generic API Key
RuleID:      generic-api-key
File:        backend/config.py
Line:        12
Commit:      0000000000000000000000000000000000000000
Author:      Developer Name
Email:       dev@tiger-analytics.com
Date:        2026-05-06T10:23:11Z
Fingerprint: backend/config.py:generic-api-key:12
Match:       DB_PASSWORD = "Tiger@ADM2024!"

1 leak found.

COMMIT REJECTED. Fix the above issues before committing.
```

Commit did NOT go through. The file is still in your working directory unchanged. Fix the secret (move to `.env`), then `git add` and commit again.

---

#### Scenario 3 — Commit blocked by lint + format

```bash
git add backend/api/routes/data.py
git commit -m "feat: add data route"
```

**What you see:**
```
🎨 Format check (black)......................................Failed
- hook id: black
- exit code: 1

would reformat backend/api/routes/data.py
Oh no! 💥 💔 💥
1 file would be reformatted, 0 files left unchanged.

🔍 Lint (ruff)...............................................Failed
- hook id: ruff
- exit code: 1

backend/api/routes/data.py:23:5: F841 Local variable `unused_var` is assigned to but never used
backend/api/routes/data.py:45:5: S608 Possible SQL injection via string-based query construction
Found 2 errors.
```

**Fix sequence:**
```bash
black .                    # auto-fix format
ruff check --fix .         # auto-fix what ruff can
# manually fix S608 (SQL injection — can't auto-fix)
git add backend/api/routes/data.py
git commit -m "feat: add data route"
```

---

### STEP 5 — The git push flow (pre-push hooks)

Pre-push runs heavier checks that would be too slow on every commit.

```bash
git push origin feature/my-new-agent
```

**What you see (clean):**
```
[pre-push hook running...]
🧠 Type check (mypy)........................................Passed
   Success: no issues found in 24 source files

Counting objects: 5, done.
Delta compression using up to 8 threads.
Writing objects: 100% (5/5), 1.23 KiB | 1.23 MiB/s, done.
To https://gitlab.yourtiger.com/adm/apis-azure.git
   3f8a2c1..a9b4d2e  feature/my-new-agent -> feature/my-new-agent
```

**What you see (mypy failure):**
```
🧠 Type check (mypy)........................................Failed
- hook id: mypy
- exit code: 1

backend/services/agent_runner.py:67: error: Incompatible return value type
(got "None", expected "Dict[str, Any]")  [return-value]

backend/api/routes/agents.py:112: error: Argument 1 to "run_agent"
has incompatible type "Optional[str]"; expected "str"  [arg-type]

Found 2 errors in 2 files (checked 24 source files)

PUSH REJECTED. Fix type errors before pushing.
```

Push did NOT reach GitLab. Fix the type errors, re-commit, then push again.

---

## PART 2 — REPOSITORY FILE SETUP

---

### STEP 6 — Files to create in each repo (do this once)

#### 6.1 `apis-azure` file structure

```
apis-azure/
├── .pre-commit-config.yaml        ← hooks config
├── .gitleaks.toml                 ← secret scan rules
├── pyproject.toml                 ← ruff/black/mypy/bandit config
├── requirements-dev.txt           ← dev dependencies
├── .env.example                   ← template (committed)
├── .env                           ← real secrets (NEVER committed)
├── .gitignore                     ← must include .env
└── .gitlab-ci.yml                 ← CI pipeline
```

#### File: `apis-azure/.pre-commit-config.yaml`

```yaml
# .pre-commit-config.yaml
# ─────────────────────────────────────────────────────────────────────
# SETUP: Run once after cloning:
#   pip install pre-commit
#   pre-commit install
#   pre-commit install --hook-type pre-push
# ─────────────────────────────────────────────────────────────────────

repos:

  # ── LAYER 1: SECRET SCANNING ─────────────────────────────────────
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.18.4
    hooks:
      - id: gitleaks
        name: "🔑 Secret scan (gitleaks)"
        description: "Detects secrets, passwords, tokens in committed files"

  # ── LAYER 2: FILE HYGIENE ────────────────────────────────────────
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.6.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-json
      - id: check-toml
      - id: check-merge-conflict
      - id: check-added-large-files
        args: ['--maxkb=500']
      - id: detect-private-key
      - id: no-commit-to-branch
        args: ['--branch', 'main', '--branch', 'master', '--branch', 'develop']

  # ── LAYER 3: PYTHON FORMATTING ──────────────────────────────────
  - repo: https://github.com/psf/black
    rev: 24.4.2
    hooks:
      - id: black
        name: "🎨 Format check (black)"
        language_version: python3

  # ── LAYER 4: PYTHON LINT ─────────────────────────────────────────
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.4.7
    hooks:
      - id: ruff
        name: "🔍 Lint (ruff)"
        args: [--fix]
      - id: ruff-format

  # ── LAYER 5: TYPE CHECK (pre-push only — slower) ─────────────────
  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.10.0
    hooks:
      - id: mypy
        name: "🧠 Type check (mypy)"
        stages: [pre-push]           # ← only runs on git push, not git commit
        additional_dependencies:
          - types-requests
          - types-PyYAML
          - fastapi
          - pydantic
        args:
          - --ignore-missing-imports
          - --no-strict-optional
          - --show-error-codes

  # ── LAYER 6: CUSTOM SECURITY PATTERNS ───────────────────────────
  - repo: local
    hooks:

      - id: no-verify-false
        name: "🚫 Block verify=False (CWE-295: Improper Certificate Validation)"
        language: pygrep
        entry: 'verify\s*=\s*False'
        types: [python]
        files: '\.py$'
        exclude: '(test_|_test\.py|conftest\.py)'

      - id: no-fstring-sql
        name: "🚫 Block f-string SQL (CWE-89: SQL Injection)"
        language: pygrep
        entry: 'f["\''](SELECT|INSERT|UPDATE|DELETE|DROP|CREATE|ALTER|TRUNCATE)'
        types: [python]
        files: '\.py$'
        exclude: '(test_|_test\.py)'

      - id: no-hardcoded-db-passwords
        name: "🚫 Block hardcoded credentials (CWE-798)"
        language: pygrep
        entry: '(DB_PASSWORD|db_password|DATABASE_PASSWORD|POSTGRES_PASSWORD|SQL_PASSWORD)\s*=\s*["\x27][^"\x27$\{][^"\x27]{3,}["\x27]'
        types: [python]
        files: '\.py$'
        exclude: '(\.example|test_|_test\.py|conftest\.py)'

      - id: no-real-env-file
        name: "🚫 Block committing .env (use .env.example)"
        language: fail
        entry: "STOP: .env file must never be committed. Use .env.example with placeholder values."
        files: '^\.env$'

      - id: no-wildcard-cors
        name: "🚫 Block wildcard CORS (CWE-942)"
        language: pygrep
        entry: 'allow_origins\s*=\s*\[?\s*["\x27]\*["\x27]'
        types: [python]
        files: '\.py$'
        exclude: '(test_|_test\.py)'
```

#### File: `apis-azure/.gitleaks.toml`

```toml
title = "ADM Silver Layer — Gitleaks Configuration"

[extend]
useDefault = true    # extends gitleaks built-in rules (AWS, Azure, GCP keys etc.)

# ── CUSTOM ADM RULES ────────────────────────────────────────────────

[[rules]]
id = "adm-db-connection-string"
description = "Database connection string with embedded credentials"
regex = '''(mssql|postgresql|postgres|mongodb|mysql|sqlite):\/\/[^:]+:[^@\s]+@'''
severity = "CRITICAL"
tags = ["database", "credentials", "adm"]

[[rules]]
id = "azure-storage-account-key"
description = "Azure Storage Account connection string"
regex = '''DefaultEndpointsProtocol=https;AccountName=[^;]+;AccountKey=[A-Za-z0-9+\/=]{60,}'''
severity = "CRITICAL"
tags = ["azure", "storage", "adm"]

[[rules]]
id = "azure-function-key"
description = "Azure Function App key"
regex = '''["\x27]?(x-functions-key|Ocp-Apim-Subscription-Key)["\x27]?\s*[:=]\s*["\x27][A-Za-z0-9\/+]{20,}["\x27]'''
severity = "HIGH"
tags = ["azure", "functions", "adm"]

[[rules]]
id = "hardcoded-localhost-url"
description = "Hardcoded localhost URL with port (config drift risk)"
regex = '''(http:\/\/localhost:\d{4,5}|http:\/\/127\.0\.0\.1:\d{4,5})'''
severity = "MEDIUM"
tags = ["config-drift", "adm"]

# ── GLOBAL ALLOWLIST ─────────────────────────────────────────────────

[allowlist]
description = "Files that are safe to have pattern matches"
files = [
  '''\.env\.example$''',
  '''README(\.md)?$''',
  '''CHANGELOG(\.md)?$''',
  '''.*_test\.py$''',
  '''test_.*\.py$''',
  '''conftest\.py$''',
  '''\.gitlab-ci\.yml$''',
]
regexes = [
  '''your-.*-here''',         # placeholder values in .env.example
  '''example\.com''',
  '''localhost:PORT''',
]
```

#### File: `apis-azure/pyproject.toml`

```toml
[tool.ruff]
target-version = "py311"
line-length = 100
select = [
  "E",   # pycodestyle errors
  "W",   # pycodestyle warnings
  "F",   # pyflakes
  "I",   # isort
  "B",   # flake8-bugbear (common bugs)
  "S",   # flake8-bandit (security — SQL injection, subprocess, etc.)
  "UP",  # pyupgrade
  "C90", # mccabe complexity
]
ignore = [
  "S101",  # assert — ok in tests
  "B008",  # function calls in defaults — FastAPI Depends() pattern
  "S104",  # binding to 0.0.0.0 — ok for containerized services
]
exclude = [".venv", "venv", "__pycache__", "alembic", "migrations"]

[tool.ruff.per-file-ignores]
"tests/*" = ["S", "B", "E501"]
"**/conftest.py" = ["S"]

[tool.ruff.mccabe]
max-complexity = 12

[tool.black]
line-length = 100
target-version = ["py311"]
exclude = '''
/(
    \.venv
  | venv
  | __pycache__
  | alembic
)/
'''

[tool.mypy]
python_version = "3.11"
ignore_missing_imports = true
warn_return_any = true
warn_unused_configs = true
warn_unused_ignores = true
no_implicit_optional = true
exclude = ["venv", ".venv", "tests", "alembic"]

[tool.bandit]
exclude_dirs = ["tests", "venv", ".venv", "alembic"]
skips = ["B101"]
severity = "medium"
confidence = "medium"

[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]
python_files = ["test_*.py", "*_test.py"]
```

#### File: `apis-azure/.env.example`

```bash
# Copy this to .env and fill in real values
# NEVER commit .env — only commit .env.example

# ── DATABASE ──────────────────────────────────────────────────────
DATABASE_URL=postgresql://user:password@host:5432/dbname
DB_HOST=your-db-host.postgres.database.azure.com
DB_PORT=5432
DB_NAME=adm_silver
DB_USER=your-db-user
DB_PASSWORD=your-db-password-here

# ── AZURE ─────────────────────────────────────────────────────────
AZURE_CLIENT_ID=your-client-id-here
AZURE_CLIENT_SECRET=your-client-secret-here
AZURE_TENANT_ID=your-tenant-id-here
AZURE_STORAGE_CONNECTION_STRING=your-connection-string-here

# ── AGENTS ────────────────────────────────────────────────────────
AGENT_BASE_URL=https://your-function-app.azurewebsites.net
AGENT_FUNCTION_KEY=your-function-key-here

# ── APP ───────────────────────────────────────────────────────────
ENVIRONMENT=development
DEBUG=false
SECRET_KEY=your-secret-key-here
```

#### File: `apis-azure/.gitignore` (key additions)

```
# Environment files — NEVER commit these
.env
.env.local
.env.*.local

# Python
.venv/
venv/
__pycache__/
*.pyc
*.pyo
.mypy_cache/
.ruff_cache/
.pytest_cache/
htmlcov/
.coverage
coverage.xml

# IDE
.vscode/
.idea/
*.swp

# CI artifacts
bandit-report.json
semgrep-report.sarif
gitleaks-report.sarif
pip-audit-report.json
```

---

## PART 3 — GITLAB CI/CD PIPELINE SETUP

---

### STEP 7 — Understanding the pipeline stages

Every MR triggers this sequence:

```
Stage 1: lint         → ruff, black, mypy, eslint, tsc
          ↓ (fails here = stops, no point running security)
Stage 2: security     → gitleaks, bandit, semgrep, custom checks
          ↓
Stage 3: test         → pytest with coverage gate
          ↓
Stage 4: audit        → pip-audit / npm audit (dependency CVEs)
          ↓
Stage 5: sonarqube    → SonarQube analysis + quality gate
          ↓
        MR ALLOWED TO MERGE  ← only if ALL stages pass
```

---

### STEP 8 — `apis-azure/.gitlab-ci.yml` (complete)

```yaml
# ─────────────────────────────────────────────────────────────────────────────
# apis-azure GitLab CI Pipeline
# Triggers: every MR, every push to main/develop
# All jobs use allow_failure: false — any failure blocks the MR
# ─────────────────────────────────────────────────────────────────────────────

stages:
  - lint
  - security
  - test
  - audit
  - sonarqube

variables:
  PIP_CACHE_DIR: "$CI_PROJECT_DIR/.pip-cache"
  SONAR_USER_HOME: "${CI_PROJECT_DIR}/.sonar"
  GIT_DEPTH: "0"    # full clone needed for sonar + gitleaks history scan

default:
  image: python:3.11-slim
  cache:
    - key: "pip-$CI_COMMIT_REF_SLUG"
      paths:
        - .pip-cache/
    - key: "sonar-$CI_COMMIT_REF_SLUG"
      paths:
        - .sonar/cache

# ════════════════════════════════════════════════════════════════════════════
# STAGE 1 — LINT
# ════════════════════════════════════════════════════════════════════════════

ruff-lint:
  stage: lint
  script:
    - pip install ruff --quiet
    - echo "Running ruff lint check..."
    - ruff check . --output-format=gitlab
  artifacts:
    reports:
      codequality: gl-code-quality-report.json
    when: always
    expire_in: 1 week
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH =~ /^(main|develop)$/'
  allow_failure: false

black-format-check:
  stage: lint
  script:
    - pip install black --quiet
    - echo "Checking code formatting with black..."
    - black --check --diff .
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
  allow_failure: false

mypy-type-check:
  stage: lint
  script:
    - pip install mypy types-requests types-PyYAML pydantic fastapi --quiet
    - pip install -r requirements.txt --quiet
    - echo "Running mypy type check..."
    - mypy .
        --ignore-missing-imports
        --no-strict-optional
        --show-error-codes
        --junit-xml=mypy-report.xml
  artifacts:
    reports:
      junit: mypy-report.xml
    when: always
    expire_in: 1 week
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
  allow_failure: false

# ════════════════════════════════════════════════════════════════════════════
# STAGE 2 — SECURITY
# ════════════════════════════════════════════════════════════════════════════

secret-scan:
  stage: security
  image:
    name: zricethezav/gitleaks:latest
    entrypoint: [""]
  script:
    - echo "Scanning for secrets and credentials..."
    - gitleaks detect
        --source .
        --config .gitleaks.toml
        --report-format sarif
        --report-path gitleaks-report.sarif
        --verbose
        --exit-code 1
  artifacts:
    when: always
    paths:
      - gitleaks-report.sarif
    reports:
      sast: gitleaks-report.sarif
    expire_in: 1 month
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH =~ /^(main|develop)$/'
  allow_failure: false

bandit-sast:
  stage: security
  script:
    - pip install "bandit[toml]" --quiet
    - echo "Running Bandit SAST (Python security analysis)..."
    - bandit
        --recursive .
        --configfile pyproject.toml
        --format json
        --output bandit-report.json
        --severity-level medium
        --confidence-level medium
        --exit-zero
    # Now check report — fail if any HIGH or CRITICAL issues
    - pip install jq --quiet || apt-get install -y jq -q
    - |
      HIGH_COUNT=$(python3 -c "
      import json, sys
      with open('bandit-report.json') as f:
          data = json.load(f)
      issues = data.get('results', [])
      high = [i for i in issues if i['issue_severity'] in ('HIGH', 'CRITICAL')]
      print(len(high))
      for i in high:
          print(f\"  {i['filename']}:{i['line_number']}: [{i['issue_severity']}] {i['issue_text']}\", file=sys.stderr)
      ")
      if [ "$HIGH_COUNT" -gt "0" ]; then
        echo "❌ Bandit found $HIGH_COUNT HIGH/CRITICAL severity issues. Fix before merging."
        bandit --recursive . --configfile pyproject.toml --severity-level high
        exit 1
      fi
      echo "✅ Bandit: No HIGH/CRITICAL issues found"
  artifacts:
    when: always
    paths:
      - bandit-report.json
    expire_in: 1 month
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
  allow_failure: false

semgrep-sast:
  stage: security
  image: returntocorp/semgrep:latest
  variables:
    SEMGREP_APP_TOKEN: $SEMGREP_APP_TOKEN   # optional — set in GitLab CI variables if using semgrep cloud
  script:
    - echo "Running Semgrep SAST..."
    - semgrep scan
        --config=p/python
        --config=p/fastapi
        --config=p/owasp-top-ten
        --config=p/secrets
        --error
        --sarif
        --output=semgrep-report.sarif
        --exclude=tests
        --exclude=venv
        --exclude=.venv
        .
  artifacts:
    when: always
    paths:
      - semgrep-report.sarif
    reports:
      sast: semgrep-report.sarif
    expire_in: 1 month
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
  allow_failure: false

custom-security-patterns:
  stage: security
  script:
    - echo "=== Custom ADM Security Pattern Checks ==="
    - FAILED=0

    - echo ""
    - echo "── Check 1: verify=False (CWE-295) ──"
    - |
      RESULT=$(grep -rn "verify\s*=\s*False" --include="*.py" \
        --exclude-dir=tests --exclude-dir=venv --exclude-dir=.venv . || true)
      if [ -n "$RESULT" ]; then
        echo "❌ FAIL: verify=False detected (disables TLS certificate validation)"
        echo "$RESULT"
        FAILED=1
      else
        echo "✅ PASS"
      fi

    - echo ""
    - echo "── Check 2: f-string SQL (CWE-89) ──"
    - |
      RESULT=$(grep -Prn 'f["\x27](SELECT|INSERT|UPDATE|DELETE|DROP|TRUNCATE|ALTER)' \
        --include="*.py" --exclude-dir=tests --exclude-dir=venv . || true)
      if [ -n "$RESULT" ]; then
        echo "❌ FAIL: f-string SQL detected (SQL Injection risk)"
        echo "$RESULT"
        FAILED=1
      else
        echo "✅ PASS"
      fi

    - echo ""
    - echo "── Check 3: Hardcoded credentials (CWE-798) ──"
    - |
      RESULT=$(grep -Prn '(DB_PASSWORD|database_password|POSTGRES_PASSWORD)\s*=\s*["\x27][^"\x27$\{]' \
        --include="*.py" --exclude-dir=tests --exclude-dir=venv . || true)
      if [ -n "$RESULT" ]; then
        echo "❌ FAIL: Possible hardcoded credential"
        echo "$RESULT"
        FAILED=1
      else
        echo "✅ PASS"
      fi

    - echo ""
    - echo "── Check 4: Wildcard CORS (CWE-942) ──"
    - |
      RESULT=$(grep -Prn 'allow_origins\s*=\s*\[?\s*["\x27]\*["\x27]' \
        --include="*.py" --exclude-dir=tests . || true)
      if [ -n "$RESULT" ]; then
        echo "❌ FAIL: Wildcard CORS (*) detected"
        echo "$RESULT"
        FAILED=1
      else
        echo "✅ PASS"
      fi

    - echo ""
    - |
      if [ "$FAILED" -eq "1" ]; then
        echo "═══════════════════════════════════════════"
        echo "❌ Custom security checks FAILED"
        echo "Fix the above issues before this MR can merge."
        echo "═══════════════════════════════════════════"
        exit 1
      else
        echo "✅ All custom security checks passed"
      fi
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
  allow_failure: false

# ════════════════════════════════════════════════════════════════════════════
# STAGE 3 — TEST
# ════════════════════════════════════════════════════════════════════════════

pytest:
  stage: test
  script:
    - pip install pytest pytest-cov pytest-asyncio httpx --quiet
    - pip install -r requirements.txt --quiet
    - echo "Running tests with coverage..."
    - pytest tests/
        --cov=backend
        --cov-report=xml:coverage.xml
        --cov-report=term-missing
        --cov-report=html:htmlcov
        --cov-fail-under=60
        --junit-xml=pytest-report.xml
        -v
  coverage: '/(?i)total.*? (100(?:\.0+)?\%|[1-9]?\d(?:\.\d+)?\%)$/'
  artifacts:
    when: always
    reports:
      coverage_report:
        coverage_format: cobertura
        path: coverage.xml
      junit: pytest-report.xml
    paths:
      - htmlcov/
      - coverage.xml
    expire_in: 1 month
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
  allow_failure: false

# ════════════════════════════════════════════════════════════════════════════
# STAGE 4 — AUDIT
# ════════════════════════════════════════════════════════════════════════════

pip-audit:
  stage: audit
  script:
    - pip install pip-audit --quiet
    - echo "Auditing Python dependencies for known CVEs..."
    - pip-audit
        --requirement requirements.txt
        --format json
        --output pip-audit-report.json
        --vulnerability-service pypi
    - echo "Audit complete. Checking severity..."
    - |
      python3 -c "
      import json, sys
      with open('pip-audit-report.json') as f:
          data = json.load(f)
      vulns = data.get('dependencies', [])
      critical = []
      for dep in vulns:
          for v in dep.get('vulns', []):
              if v.get('fix_versions'):
                  critical.append(f\"{dep['name']} {dep['version']}: {v['id']} — {v['description'][:80]}\")
      if critical:
          print('❌ Vulnerabilities with available fixes:')
          for c in critical: print(f'  {c}')
          sys.exit(1)
      print(f'✅ {len(vulns)} packages audited. No critical vulnerabilities with available fixes.')
      "
  artifacts:
    when: always
    paths:
      - pip-audit-report.json
    expire_in: 1 month
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH == "main"'
      when: always
  allow_failure: false

# Scheduled job — runs CVE audit on main daily even without MRs
pip-audit-scheduled:
  stage: audit
  script:
    - pip install pip-audit --quiet
    - pip-audit --requirement requirements.txt --fail-on-severity high
  rules:
    - if: '$CI_PIPELINE_SOURCE == "schedule"'
  allow_failure: false

# ════════════════════════════════════════════════════════════════════════════
# STAGE 5 — SONARQUBE
# ════════════════════════════════════════════════════════════════════════════

sonarqube-analysis:
  stage: sonarqube
  image:
    name: sonarsource/sonar-scanner-cli:latest
    entrypoint: [""]
  variables:
    SONAR_TOKEN: $SONAR_TOKEN                    # set in GitLab CI/CD variables
    SONAR_HOST_URL: $SONAR_HOST_URL              # e.g. https://sonar.yourtiger.com
  script:
    - echo "Running SonarQube analysis..."
    - sonar-scanner
        -Dsonar.projectKey=adm-apis-azure
        -Dsonar.projectName="ADM APIs Azure"
        -Dsonar.sources=backend
        -Dsonar.tests=tests
        -Dsonar.python.coverage.reportPaths=coverage.xml
        -Dsonar.python.bandit.reportPaths=bandit-report.json
        -Dsonar.host.url=$SONAR_HOST_URL
        -Dsonar.token=$SONAR_TOKEN
        -Dsonar.qualitygate.wait=true
  dependencies:
    - pytest
    - bandit-sast
  allow_failure: false
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH =~ /^(main|develop)$/'
```

---

### STEP 9 — `adm-foundation-layer-agents-azure/.gitlab-ci.yml` (complete)

```yaml
stages:
  - lint
  - security
  - test
  - audit
  - sonarqube

variables:
  PIP_CACHE_DIR: "$CI_PROJECT_DIR/.pip-cache"
  GIT_DEPTH: "0"

default:
  image: python:3.11-slim
  cache:
    key: "pip-agents-$CI_COMMIT_REF_SLUG"
    paths:
      - .pip-cache/

# ════════════════════════════════════════════════════════════════════════════
# STAGE 1 — LINT
# ════════════════════════════════════════════════════════════════════════════

ruff-lint:
  stage: lint
  script:
    - pip install ruff --quiet
    - ruff check . --output-format=gitlab
  allow_failure: false
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH =~ /^(main|develop)$/'

mypy-type-check:
  stage: lint
  script:
    - pip install mypy types-requests azure-functions --quiet
    - pip install -r requirements.txt --quiet
    - mypy . --ignore-missing-imports --show-error-codes
  allow_failure: false
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'

# ════════════════════════════════════════════════════════════════════════════
# STAGE 2 — SECURITY
# ════════════════════════════════════════════════════════════════════════════

secret-scan:
  stage: security
  image:
    name: zricethezav/gitleaks:latest
    entrypoint: [""]
  script:
    - gitleaks detect --source . --config .gitleaks.toml --exit-code 1 --verbose
  artifacts:
    when: always
    paths:
      - gitleaks-report.sarif
  allow_failure: false
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH =~ /^(main|develop)$/'

bandit-sast:
  stage: security
  script:
    - pip install "bandit[toml]" --quiet
    - bandit -r . --configfile pyproject.toml -f json -o bandit-report.json --exit-zero
    - |
      python3 -c "
      import json, sys
      data = json.load(open('bandit-report.json'))
      high = [i for i in data.get('results',[]) if i['issue_severity'] in ('HIGH','CRITICAL')]
      if high:
          [print(f\"  {i['filename']}:{i['line_number']}: {i['issue_text']}\") for i in high]
          sys.exit(1)
      print(f'✅ {len(data[\"results\"])} issues checked. No HIGH/CRITICAL.')
      "
  artifacts:
    when: always
    paths:
      - bandit-report.json
  allow_failure: false
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'

agent-specific-checks:
  stage: security
  script:
    - echo "=== ADM Agents-specific Security Checks ==="
    - FAILED=0

    - echo "── Check: verify=False ──"
    - |
      if grep -rn "verify=False" --include="*.py" --exclude-dir=tests .; then
        echo "❌ verify=False found"; FAILED=1
      else echo "✅ PASS"; fi

    - echo "── Check: Real .env not tracked ──"
    - |
      if git ls-files | grep -E '^\.env$'; then
        echo "❌ .env is tracked by git!"; FAILED=1
      else echo "✅ PASS"; fi

    - echo "── Check: Connector duplication ──"
    - |
      COUNT=$(find . -name "VectorDatabaseConnectors.py" -not -path "*/shared/*" | wc -l)
      if [ "$COUNT" -gt "1" ]; then
        echo "❌ $COUNT copies of VectorDatabaseConnectors.py found:"
        find . -name "VectorDatabaseConnectors.py" -not -path "*/shared/*"
        echo "Move to shared/ package. Duplication is banned per ADM standards."
        FAILED=1
      else echo "✅ PASS (connector files: $COUNT)"; fi

    - echo "── Check: f-string SQL ──"
    - |
      if grep -Prn 'f["\x27](SELECT|INSERT|UPDATE|DELETE)' --include="*.py" .; then
        echo "❌ f-string SQL found"; FAILED=1
      else echo "✅ PASS"; fi

    - |
      [ "$FAILED" -eq "1" ] && { echo "❌ Agent security checks FAILED"; exit 1; } \
        || echo "✅ All agent security checks passed"
  allow_failure: false
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'

# ════════════════════════════════════════════════════════════════════════════
# STAGE 3 — TEST
# ════════════════════════════════════════════════════════════════════════════

pytest:
  stage: test
  script:
    - pip install pytest pytest-cov pytest-asyncio --quiet
    - pip install -r requirements.txt --quiet
    - pytest tests/ --cov=. --cov-report=xml --cov-fail-under=50 --junit-xml=pytest-report.xml -v
  coverage: '/(?i)total.*? (100(?:\.0+)?\%|[1-9]?\d(?:\.\d+)?\%)$/'
  artifacts:
    when: always
    reports:
      coverage_report:
        coverage_format: cobertura
        path: coverage.xml
      junit: pytest-report.xml
  allow_failure: false
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'

# ════════════════════════════════════════════════════════════════════════════
# STAGE 4 — AUDIT
# ════════════════════════════════════════════════════════════════════════════

pip-audit:
  stage: audit
  script:
    - pip install pip-audit --quiet
    - pip-audit -r requirements.txt --fail-on-severity high
  allow_failure: false
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'

# ════════════════════════════════════════════════════════════════════════════
# STAGE 5 — SONARQUBE
# ════════════════════════════════════════════════════════════════════════════

sonarqube-analysis:
  stage: sonarqube
  image:
    name: sonarsource/sonar-scanner-cli:latest
    entrypoint: [""]
  script:
    - sonar-scanner
        -Dsonar.projectKey=adm-agents-azure
        -Dsonar.projectName="ADM Agents Azure"
        -Dsonar.sources=.
        -Dsonar.exclusions=tests/**,venv/**,.venv/**
        -Dsonar.tests=tests
        -Dsonar.python.coverage.reportPaths=coverage.xml
        -Dsonar.python.bandit.reportPaths=bandit-report.json
        -Dsonar.host.url=$SONAR_HOST_URL
        -Dsonar.token=$SONAR_TOKEN
        -Dsonar.qualitygate.wait=true
  dependencies:
    - pytest
    - bandit-sast
  allow_failure: false
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH =~ /^(main|develop)$/'
```

---

### STEP 10 — `apps-azure/.gitlab-ci.yml` (complete — React/TypeScript)

```yaml
stages:
  - lint
  - security
  - test
  - audit
  - sonarqube

variables:
  NODE_VERSION: "20"
  SONAR_USER_HOME: "${CI_PROJECT_DIR}/.sonar"
  GIT_DEPTH: "0"

default:
  image: node:20-slim
  cache:
    - key: "npm-$CI_COMMIT_REF_SLUG"
      paths:
        - node_modules/
        - .npm/
    - key: "sonar-$CI_COMMIT_REF_SLUG"
      paths:
        - .sonar/cache

# ════════════════════════════════════════════════════════════════════════════
# STAGE 1 — LINT
# ════════════════════════════════════════════════════════════════════════════

eslint:
  stage: lint
  script:
    - npm ci --cache .npm --prefer-offline --quiet
    - echo "Running ESLint..."
    - npx eslint . --ext .ts,.tsx,.js,.jsx --max-warnings 0 --format stylish
  allow_failure: false
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH =~ /^(main|develop)$/'

typescript-check:
  stage: lint
  script:
    - npm ci --cache .npm --prefer-offline --quiet
    - echo "Running TypeScript type check..."
    - npx tsc --noEmit
  allow_failure: false
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'

prettier-check:
  stage: lint
  script:
    - npm ci --cache .npm --prefer-offline --quiet
    - echo "Checking code formatting with Prettier..."
    - npx prettier --check "src/**/*.{ts,tsx,js,jsx,css,json}"
  allow_failure: false
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'

# ════════════════════════════════════════════════════════════════════════════
# STAGE 2 — SECURITY
# ════════════════════════════════════════════════════════════════════════════

secret-scan:
  stage: security
  image:
    name: zricethezav/gitleaks:latest
    entrypoint: [""]
  script:
    - gitleaks detect --source . --config .gitleaks.toml --exit-code 1 --verbose
  allow_failure: false
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH =~ /^(main|develop)$/'

semgrep-frontend:
  stage: security
  image: returntocorp/semgrep:latest
  script:
    - semgrep scan
        --config=p/typescript
        --config=p/react
        --config=p/secrets
        --config=p/owasp-top-ten
        --error
        --sarif
        --output=semgrep-report.sarif
        --exclude=node_modules
        --exclude="*.test.*"
        --exclude="*.spec.*"
        .
  artifacts:
    when: always
    paths:
      - semgrep-report.sarif
    reports:
      sast: semgrep-report.sarif
  allow_failure: false
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'

custom-frontend-checks:
  stage: security
  script:
    - echo "=== ADM Frontend Security Checks ==="
    - FAILED=0

    - echo "── Check: localStorage banned for sensitive state ──"
    - |
      HITS=$(grep -rn "localStorage\.setItem" src/ \
        --include="*.ts" --include="*.tsx" \
        | grep -v "\.test\." | grep -v "\.spec\." || true)
      if [ -n "$HITS" ]; then
        echo "❌ localStorage.setItem detected in production code:"
        echo "$HITS"
        echo "Use React state / Zustand / backend API instead."
        FAILED=1
      else echo "✅ PASS"; fi

    - echo "── Check: No hardcoded localhost in source ──"
    - |
      HITS=$(grep -rn "http://localhost:" src/ \
        --include="*.ts" --include="*.tsx" \
        | grep -v "\.test\." | grep -v "\.spec\." || true)
      if [ -n "$HITS" ]; then
        echo "❌ Hardcoded localhost URL in source:"
        echo "$HITS"
        echo "Use VITE_API_BASE_URL environment variable."
        FAILED=1
      else echo "✅ PASS"; fi

    - echo "── Check: No hardcoded Azure production URLs ──"
    - |
      HITS=$(grep -Prn 'https?://[a-zA-Z0-9\-]+(\.azurewebsites\.net|\.azurefd\.net|\.azure\.com)' \
        src/ --include="*.ts" --include="*.tsx" \
        | grep -v "\.test\." | grep -v "\.spec\." || true)
      if [ -n "$HITS" ]; then
        echo "❌ Hardcoded Azure URL detected:"
        echo "$HITS"
        echo "Use VITE_* environment variables for all API URLs."
        FAILED=1
      else echo "✅ PASS"; fi

    - echo "── Check: No API keys in frontend code ──"
    - |
      HITS=$(grep -Prn '(api_key|apiKey|API_KEY)\s*[:=]\s*["\x27][A-Za-z0-9]{20,}' \
        src/ --include="*.ts" --include="*.tsx" | grep -v "\.test\." || true)
      if [ -n "$HITS" ]; then
        echo "❌ Possible API key in frontend source:"
        echo "$HITS"
        FAILED=1
      else echo "✅ PASS"; fi

    - |
      [ "$FAILED" -eq "1" ] && { echo "❌ Frontend security checks FAILED"; exit 1; } \
        || echo "✅ All frontend security checks passed"
  allow_failure: false
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'

# ════════════════════════════════════════════════════════════════════════════
# STAGE 3 — TEST
# ════════════════════════════════════════════════════════════════════════════

vitest:
  stage: test
  script:
    - npm ci --cache .npm --prefer-offline --quiet
    - echo "Running Vitest with coverage..."
    - npx vitest run --coverage --reporter=verbose
  coverage: '/Statements\s*:\s*([\d.]+)%/'
  artifacts:
    when: always
    reports:
      coverage_report:
        coverage_format: cobertura
        path: coverage/cobertura-coverage.xml
      junit: coverage/junit-report.xml
    paths:
      - coverage/
    expire_in: 1 month
  allow_failure: false
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'

# ════════════════════════════════════════════════════════════════════════════
# STAGE 4 — AUDIT
# ════════════════════════════════════════════════════════════════════════════

npm-audit:
  stage: audit
  script:
    - npm ci --cache .npm --prefer-offline --quiet
    - echo "Auditing npm dependencies..."
    - npm audit --audit-level=high --json > npm-audit.json || true
    - |
      node -e "
      const report = require('./npm-audit.json');
      const vulns = report.metadata?.vulnerabilities || {};
      const criticalHigh = (vulns.critical || 0) + (vulns.high || 0);
      console.log('Vulnerabilities:', JSON.stringify(vulns));
      if (criticalHigh > 0) {
        console.error('❌ ' + criticalHigh + ' HIGH/CRITICAL npm vulnerabilities found');
        process.exit(1);
      }
      console.log('✅ No HIGH/CRITICAL npm vulnerabilities');
      "
  artifacts:
    when: always
    paths:
      - npm-audit.json
  allow_failure: false
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'

# ════════════════════════════════════════════════════════════════════════════
# STAGE 5 — SONARQUBE
# ════════════════════════════════════════════════════════════════════════════

sonarqube-analysis:
  stage: sonarqube
  image:
    name: sonarsource/sonar-scanner-cli:latest
    entrypoint: [""]
  script:
    - sonar-scanner
        -Dsonar.projectKey=adm-apps-azure
        -Dsonar.projectName="ADM Apps Azure"
        -Dsonar.sources=src
        -Dsonar.exclusions=node_modules/**,coverage/**,dist/**
        -Dsonar.tests=src
        -Dsonar.test.inclusions="**/*.test.ts,**/*.test.tsx,**/*.spec.ts"
        -Dsonar.javascript.lcov.reportPaths=coverage/lcov.info
        -Dsonar.host.url=$SONAR_HOST_URL
        -Dsonar.token=$SONAR_TOKEN
        -Dsonar.qualitygate.wait=true
  dependencies:
    - vitest
  allow_failure: false
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH =~ /^(main|develop)$/'
```

---

## PART 4 — SONARQUBE INTEGRATION (Detailed Setup)

---

### STEP 11 — What SonarQube adds on top of everything else

SonarQube is not just another linter. It does things the other tools don't:

| What SonarQube does | What the other tools do |
|---|---|
| Tracks code quality **trends over time** | Point-in-time checks only |
| **Quality Gate** — single pass/fail decision | Individual tool results |
| **Technical debt** calculation (time estimate) | No debt tracking |
| **Duplicate code** detection across files | Per-file checks only |
| **Cognitive complexity** per function | Cyclomatic complexity only |
| MR decoration — comments on the PR diff | No inline MR comments |
| Dashboard visible to management | CLI output only |
| **Imports bandit + coverage** reports | Separate reports |

### STEP 12 — SonarQube server setup (admin does this once)

#### 12.1 Deploy SonarQube (if not already deployed at Tiger Analytics)

First check if Tiger already has a SonarQube instance:

```bash
# Ask your DevOps/infra team:
# "Is there a SonarQube server at https://sonar.tiger-analytics.com or similar?"
```

If not, deploy via Docker (quickest for enterprise):

```bash
# docker-compose.sonarqube.yml
version: "3"
services:
  sonarqube:
    image: sonarqube:10-community
    ports:
      - "9000:9000"
    environment:
      SONAR_JDBC_URL: jdbc:postgresql://db:5432/sonarqube
      SONAR_JDBC_USERNAME: sonar
      SONAR_JDBC_PASSWORD: ${SONAR_DB_PASSWORD}
    volumes:
      - sonar_data:/opt/sonarqube/data
      - sonar_logs:/opt/sonarqube/logs
    depends_on:
      - db

  db:
    image: postgres:15
    environment:
      POSTGRES_USER: sonar
      POSTGRES_PASSWORD: ${SONAR_DB_PASSWORD}
      POSTGRES_DB: sonarqube
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  sonar_data:
  sonar_logs:
  postgres_data:
```

```bash
docker-compose -f docker-compose.sonarqube.yml up -d
```

Access at: `http://your-server:9000`  
Default login: `admin` / `admin` (change immediately)

#### 12.2 Create projects in SonarQube UI

For each repo, create a project:

1. Login to SonarQube → **Projects** → **Create Project** → **Manual**
2. Project key: `adm-apis-azure` (matches `sonar.projectKey` in your CI)
3. Project name: `ADM APIs Azure`
4. Click **Set Up**
5. Choose **With GitLab CI**
6. Follow the token generation steps

Repeat for `adm-agents-azure` and `adm-apps-azure`.

#### 12.3 Generate SonarQube tokens

In SonarQube: **My Account** → **Security** → **Generate Token**

- Name: `gitlab-ci-apis-azure`
- Type: **Project Analysis Token**
- Expiry: 365 days
- Copy the token — shown only once

#### 12.4 Add tokens to GitLab CI/CD variables

In each GitLab repo: **Settings** → **CI/CD** → **Variables** → **Add variable**

| Variable name | Value | Type | Protected | Masked |
|---|---|---|---|---|
| `SONAR_TOKEN` | `sqa_xxxxx...` | Variable | Yes | Yes |
| `SONAR_HOST_URL` | `https://sonar.yourtiger.com` | Variable | No | No |

**Protected + Masked = only runs on protected branches + hidden in logs.**

#### 12.5 Create `sonar-project.properties` in each repo root

**`apis-azure/sonar-project.properties`:**
```properties
sonar.projectKey=adm-apis-azure
sonar.projectName=ADM APIs Azure
sonar.projectVersion=1.0

# Sources
sonar.sources=backend
sonar.tests=tests
sonar.exclusions=**/venv/**,**/.venv/**,**/migrations/**,**/__pycache__/**

# Python specific
sonar.python.version=3.11
sonar.python.coverage.reportPaths=coverage.xml
sonar.python.bandit.reportPaths=bandit-report.json

# Quality Gate — uses project-level gate you configure in SonarQube UI
```

**`adm-foundation-layer-agents-azure/sonar-project.properties`:**
```properties
sonar.projectKey=adm-agents-azure
sonar.projectName=ADM Agents Azure
sonar.projectVersion=1.0

sonar.sources=.
sonar.tests=tests
sonar.exclusions=**/venv/**,**/.venv/**,**/tests/**,**/__pycache__/**

sonar.python.version=3.11
sonar.python.coverage.reportPaths=coverage.xml
sonar.python.bandit.reportPaths=bandit-report.json
```

**`apps-azure/sonar-project.properties`:**
```properties
sonar.projectKey=adm-apps-azure
sonar.projectName=ADM Apps Azure
sonar.projectVersion=1.0

sonar.sources=src
sonar.tests=src
sonar.test.inclusions=**/*.test.ts,**/*.test.tsx,**/*.spec.ts,**/*.spec.tsx
sonar.exclusions=**/node_modules/**,**/coverage/**,**/dist/**

sonar.javascript.lcov.reportPaths=coverage/lcov.info
```

#### 12.6 Configure the Quality Gate in SonarQube UI

Go to **Quality Gates** → **Create** → Name: `ADM Silver Layer Gate`

Set these conditions:

| Metric | Operator | Value |
|---|---|---|
| Coverage on new code | is less than | 60% |
| Duplicated lines on new code | is greater than | 5% |
| Maintainability rating on new code | is worse than | A |
| Reliability rating on new code | is worse than | A |
| Security rating on new code | is worse than | A |
| Security Hotspots Reviewed on new code | is less than | 80% |

Then assign this gate to all 3 projects:  
**Project** → **Project Settings** → **Quality Gate** → Select `ADM Silver Layer Gate`

#### 12.7 Connect GitLab to SonarQube (MR decoration)

This makes SonarQube **post comments directly on GitLab MRs** with analysis results.

In SonarQube: **Administration** → **DevOps Platform Integrations** → **GitLab**

Fill in:
- Configuration name: `Tiger Analytics GitLab`
- GitLab URL: `https://gitlab.yourtiger.com`
- Personal Access Token: create one in GitLab with `api` scope

In each SonarQube project: **Project Settings** → **General Settings** → **DevOps Platform Integration** → select your GitLab config → enter the GitLab project path (e.g. `adm/apis-azure`)

---

### STEP 13 — What the SonarQube pipeline output looks like

**In GitLab CI logs:**
```
Running SonarQube analysis...
INFO: Scanner configuration file: /opt/sonar-scanner/conf/sonar-scanner.properties
INFO: Project root configuration file: /builds/adm/apis-azure/sonar-project.properties
INFO: SonarScanner 5.0.1.3006
INFO: ------------- Scan ADM APIs Azure
INFO: Base dir: /builds/adm/apis-azure
INFO: Importing code coverage from: coverage.xml (cobertura format)
INFO: Parsing bandit report: bandit-report.json
INFO: CPD Executor 6 files can be analyzed
INFO: ------------- Run sensors on module ADM APIs Azure
INFO: Sensor Python Sensor [python]
INFO: Sensor Bandit Sensor [python]
INFO: ------------- Run sensors on project
INFO: SCM Publisher SCM provider for this project is: git
INFO: 142 files to be analyzed, 142/142 files analyzed
INFO: Analysis report generated in 2342ms
INFO: Analysis report compressed in 234ms, zip size=345 KB
INFO: Analysis report uploaded in 1234ms
INFO: ANALYSIS SUCCESSFUL, you can find the results at:
INFO: https://sonar.yourtiger.com/dashboard?id=adm-apis-azure
INFO: Note that you will be able to access the links to the source files
INFO: Waiting for the analysis report to be processed (max 300s)
INFO: Quality Gate status: FAILED
INFO: ██████████████████████████████████████████████
INFO: ██                QUALITY GATE               ██
INFO: ██                   FAILED                  ██
INFO: ██████████████████████████████████████████████
INFO: Quality Gate conditions:
INFO:   ✗ Coverage on New Code (45.2%) < 60% [FAILED]
INFO:   ✓ Security Rating on New Code: A [PASSED]
INFO:   ✓ Reliability Rating on New Code: A [PASSED]
INFO:   ✓ Duplicated Lines: 2.1% < 5% [PASSED]
ERROR: SONAR_SCANNER_OPTS: Quality Gate failed - MR blocked
```

**On the GitLab MR (decoration):**

SonarQube automatically posts a comment like:
```
SonarQube Quality Gate: ❌ FAILED

Coverage on new code: 45.2% (required: 60%)
  - backend/api/routes/new_agent.py: 0% coverage (23 uncovered lines)

→ View full report: https://sonar.yourtiger.com/...
```

---

## PART 5 — WHAT DEVELOPERS SEE AT EACH TRIGGER POINT (Summary)

### Trigger 1: `git commit` — Expected outputs

| Scenario | Duration | Output |
|---|---|---|
| All clean | ~5–8s | All green ✅, commit accepted |
| Secret found | ~3s | gitleaks shows file:line + match, commit REJECTED |
| Format error | ~2s | black shows which files, commit REJECTED |
| Lint error | ~3s | ruff shows file:line:rule, commit REJECTED |
| verify=False | ~1s | pygrep shows file:line, commit REJECTED |
| .env committed | instant | fail hook triggers, commit REJECTED |

### Trigger 2: `git push` — Expected outputs

| Scenario | Duration | Output |
|---|---|---|
| Type check passes | ~15–30s | mypy "Success: no issues", push accepted |
| Type errors | ~15–30s | mypy shows file:line:error, push REJECTED |

### Trigger 3: GitLab MR Pipeline — Expected outputs

| Stage | Job | Duration | Pass | Fail |
|---|---|---|---|---|
| lint | ruff-lint | ~30s | "✅ All checks passed" | File:line:rule list, pipeline fails |
| lint | black-check | ~20s | "All done! ✨" | "would reformat X files", fails |
| lint | mypy | ~60s | "Success: no issues" | Error list with codes, fails |
| security | gitleaks | ~45s | "No leaks found" | Finding dump with file:line, HARD FAIL |
| security | bandit | ~30s | "No HIGH/CRITICAL issues" | Issue list with CWE, fails |
| security | semgrep | ~90s | "Ran N rules" | Rule violations with fix suggestions, fails |
| security | custom-checks | ~15s | "✅ All custom checks passed" | Which check failed + file:line, fails |
| test | pytest | ~60–120s | "X passed, Y warnings" + coverage% | Test names that failed + coverage below threshold |
| audit | pip-audit | ~30s | "N packages audited, 0 vulns" | CVE ID + package + severity, fails |
| sonarqube | sonar-scanner | ~2–3min | "Quality Gate: PASSED ✅" | Which condition failed + value vs threshold |

---

## PART 6 — TEAM ROLLOUT CHECKLIST

### Admin / You do once:
- [ ] Deploy SonarQube server (or confirm Tiger has one)
- [ ] Create 3 SonarQube projects + custom Quality Gate
- [ ] Generate SonarQube tokens for each project
- [ ] Add `SONAR_TOKEN` and `SONAR_HOST_URL` to GitLab CI variables in all 3 repos
- [ ] Set `main` as protected branch: "Pipelines must succeed" enabled
- [ ] Enable "All threads must be resolved" in MR settings
- [ ] Commit `.pre-commit-config.yaml`, `.gitleaks.toml`, `pyproject.toml`, `.gitlab-ci.yml` to all repos
- [ ] Commit `.env.example` for all repos, ensure `.env` is in `.gitignore`

### Each developer does once per repo:
```bash
git pull
pip install pre-commit
pre-commit install
pre-commit install --hook-type pre-push
pip install -r requirements-dev.txt
cp .env.example .env
# Fill in .env with real values from Azure Key Vault / team lead
```

### Verify it's working:
```bash
# Should show all hooks installed
pre-commit run --all-files

# Should reject this:
echo 'DB_PASSWORD = "test123!"' > /tmp/test.py
git add /tmp/test.py && git commit -m "test"
# Expected: REJECTED by gitleaks + hardcoded password hook
```

---

## QUICK REFERENCE — Fixing Common Failures

| Error message | Cause | Fix command |
|---|---|---|
| `would reformat X.py` | Black format | `black .` |
| `F401 imported but unused` | Ruff | `ruff check --fix .` |
| `S608 SQL injection` | f-string SQL | Manual fix — parameterized query |
| `verify=False found` | TLS disabled | Manual fix — remove verify=False |
| `generic-api-key found` | Hardcoded secret | Move to .env, rotate credential |
| `Incompatible type` | mypy | Manual fix — add type annotation |
| `would not commit to branch` | Push to protected | Use feature branch |
| `Coverage X% < 60%` | Low test coverage | Write tests for new code |
| `Quality Gate: FAILED` | SonarQube gate | Check SonarQube dashboard for detail |
