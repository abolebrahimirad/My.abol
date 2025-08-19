name: Universal CI

on:
  push:
    branches: [ "**" ]
  pull_request:
    branches: [ "**" ]

permissions:
  contents: read
  security-events: write
  pull-requests: read

jobs:
  secrets-scan:
    name: Secret Scan (Gitleaks)
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      - name: Run Gitleaks
        uses: gitleaks/gitleaks-action@v2
        with:
          args: detect --no-git -v
  dependency-review:
    if: github.event_name == 'pull_request'
    name: Dependency Review (PR only)
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      - name: Dependency Review
        uses: actions/dependency-review-action@v4
  node:
    name: Node.js CI
    runs-on: ubuntu-latest
    if: hashFiles('**/package.json') != ''
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: 'lts/*'
          cache: npm
      - name: Install
        run: |
          if [ -f package-lock.json ]; then
            npm ci
          else
            npm install
          fi
      - name: Lint (if present)
        run: npm run -s lint --if-present
      - name: Build (if present)
        run: npm run -s build --if-present
      - name: Test (if present)
        env:
          CI: true
        run: npm run -s test --if-present
  python:
    name: Python CI
    runs-on: ubuntu-latest
    if: |
      hashFiles('**/pyproject.toml') != '' || hashFiles('**/requirements.txt') != ''
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      - name: Setup Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'
          cache: 'pip'
      - name: Install deps
        run: |
          if [ -f requirements.txt ]; then
            pip install -r requirements.txt
          fi
          if [ -f pyproject.toml ]; then
            pip install . || pip install -e . || true
          fi
          pip install pytest || true
          pip install ruff || true
      - name: Lint (ruff if available)
        run: |
          if command -v ruff >/dev/null 2>&1; then
            ruff check .
          else
            echo "ruff not installed; skipping lint"
          fi
      - name: Test (pytest if available)
        run: |
          if command -v pytest >/dev/null 2>&1; then
            pytest -q || (echo "pytest failed"; exit 1)
          else
            echo "pytest not installed; skipping tests"
          fi


