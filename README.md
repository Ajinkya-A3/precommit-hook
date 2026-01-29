# 🔐 Pre-commit + Gitleaks Setup (Standardized)

This repository uses **pre-commit** with **Gitleaks** to prevent secrets (API keys, tokens, credentials) from being committed to Git.

The setup follows **industry best practices**:
- Local pre-commit hooks → prevent accidents
- CI scanning → hard enforcement
- CI-controlled updates → safe & consistent upgrades

---

## 🧱 Architecture Overview

```
Developer Machine
 └─ pre-commit (local hook)
     └─ gitleaks (staged files)

CI Pipeline
 ├─ gitleaks scan (full repo, enforced)
 └─ pre-commit autoupdate (weekly, controlled)
 ```

---

## 1️⃣ Prerequisites (Developer Machine)

- Python 3.8+
- pip
- Git

Install pre-commit (one-time per machine):

```bash
pip install pre-commit
```

---

## 2️⃣ Pre-commit Configuration (Repository)

Each repository must contain `.pre-commit-config.yaml`.

```yaml
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.30.0
    hooks:
      - id: gitleaks
```

### Why the version (`rev`) is fixed
- Ensures reproducible behavior
- Prevents unexpected breaking changes
- Required by pre-commit by design

Versions are updated only via CI.

---

## 3️⃣ Local Setup (One-Time per Repo)

Run this inside the repository:

```bash
pre-commit install
```

This installs the Git hook into:
```
.git/hooks/pre-commit
```

---

## 4️⃣ Automated Local Setup (Bash Script)

### `scripts/setup-precommit.sh`

```bash
#!/usr/bin/env bash
set -e

echo "🔍 Checking Python..."
python --version >/dev/null 2>&1 || {
  echo "❌ Python is required"
  exit 1
}

echo "📦 Installing pre-commit..."
pip install --quiet pre-commit

CONFIG_FILE=".pre-commit-config.yaml"

if [ ! -f "$CONFIG_FILE" ]; then
  echo "📝 Creating .pre-commit-config.yaml"
  cat <<EOF > $CONFIG_FILE
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.30.0
    hooks:
      - id: gitleaks
EOF
else
  echo "✅ .pre-commit-config.yaml already exists"
fi

echo "🔗 Installing pre-commit hook..."
pre-commit install

echo "✅ Pre-commit + Gitleaks setup complete"

```

### `Run the script`

```
chmod +x scripts/setup-precommit.sh
./scripts/setup-precommit.sh

```
✔ Safe to re-run 

✔ Does not overwrite existing config

---

## 5️⃣ CI #1 — Enforce Gitleaks on Every PR / Push

### `.github/workflows/gitleaks.yml`

```yaml
name: Gitleaks Scan

on:
  pull_request:
  push:
    branches:
      - main
      - master

jobs:
  gitleaks:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - uses: gitleaks/gitleaks-action@v2
        with:
          args: detect --redact --verbose
```

---

## 6️⃣ CI #2 — Auto-Update Pre-commit Versions (Weekly)

### `.github/workflows/precommit-autoupdate.yml`

```yaml
name: Pre-commit Auto Update

on:
  schedule:
    - cron: "0 3 * * 1"
  workflow_dispatch:

jobs:
  autoupdate:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: "3.11"

      - run: pip install pre-commit
      - run: pre-commit autoupdate

      - run: |
          if git status --porcelain | grep .; then
            git config user.name "ci-bot"
            git config user.email "ci-bot@users.noreply.github.com"
            git add .pre-commit-config.yaml
            git commit -m "chore: update pre-commit hooks"
            git push
          else
            echo "No updates found"
          fi
```

---

## 7️⃣ How Updates Reach Developers

1. CI updates `rev:` in `.pre-commit-config.yaml`
2. Change is merged
3. Developer pulls latest code
4. On next commit, pre-commit auto-downloads the new version

---

## 8️⃣ Responsibilities

### Developers
- Run `pre-commit install` once per repo
- Commit normally
- Do not run `pre-commit autoupdate`

### CI / Maintainers
- Control hook versions
- Run autoupdate
- Enforce scans

---

## ✅ Summary

- `.pre-commit-config.yaml` lives in every repo
- Pre-commit prevents leaks locally
- CI enforces security
- CI manages updates
