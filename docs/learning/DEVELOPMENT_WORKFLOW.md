# Calibre Development Workflow

> **For React developers**: This guide covers professional development workflows similar to working on large React projects. Think of Git workflows like feature branches in Next.js projects, code reviews like PR reviews on GitHub, and CI/CD like Vercel deployments.

---

## Table of Contents

1. [Overview](#overview)
2. [Development Environment Setup](#development-environment-setup)
3. [Git Workflow](#git-workflow)
4. [Branch Management](#branch-management)
5. [Commit Guidelines](#commit-guidelines)
6. [Code Review Process](#code-review-process)
7. [Testing Workflow](#testing-workflow)
8. [Continuous Integration](#continuous-integration)
9. [Release Process](#release-process)
10. [Documentation Workflow](#documentation-workflow)
11. [Collaboration Best Practices](#collaboration-best-practices)

---

## Overview

Calibre follows a **professional open-source workflow** similar to large projects like React, VS Code, or Django:

### Development Cycle

```
1. Setup           → Install dependencies, configure environment
2. Create Branch   → Feature/bugfix branch from master
3. Develop         → Write code, add tests
4. Test Locally    → Run tests, lint, type check
5. Commit          → Follow commit message conventions
6. Push            → Push to your fork
7. Pull Request    → Create PR, request review
8. Code Review     → Address feedback, update PR
9. Merge           → Maintainer merges to master
10. Deploy         → CI builds and releases
```

**React equivalent**: Similar to contributing to Next.js or Create React App

---

## Development Environment Setup

### 1. Fork and Clone

```bash
# Fork on GitHub
# Visit: https://github.com/kovidgoyal/calibre
# Click "Fork"

# Clone your fork
git clone https://github.com/YOUR_USERNAME/calibre.git
cd calibre

# Add upstream remote
git remote add upstream https://github.com/kovidgoyal/calibre.git

# Verify remotes
git remote -v
# origin    https://github.com/YOUR_USERNAME/calibre.git (fetch)
# origin    https://github.com/YOUR_USERNAME/calibre.git (push)
# upstream  https://github.com/kovidgoyal/calibre.git (fetch)
# upstream  https://github.com/kovidgoyal/calibre.git (push)
```

### 2. Install Dependencies

```bash
# Create virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install Calibre in development mode
pip install -e .

# Install development dependencies
pip install -r requirements-dev.txt

# Install pre-commit hooks
pip install pre-commit
pre-commit install
```

### 3. Configure Git

```bash
# Set your identity
git config user.name "Your Name"
git config user.email "your.email@example.com"

# Enable autostash (useful for pull --rebase)
git config --global rebase.autoStash true

# Use better diff algorithm
git config --global diff.algorithm histogram

# Set default branch name
git config --global init.defaultBranch main
```

### 4. IDE Setup

**VS Code** (`.vscode/settings.json`):

```json
{
  "python.defaultInterpreterPath": "${workspaceFolder}/venv/bin/python",
  "python.linting.enabled": true,
  "python.linting.pylintEnabled": false,
  "python.linting.flake8Enabled": true,
  "python.formatting.provider": "black",
  "python.testing.pytestEnabled": true,
  "python.testing.unittestEnabled": false,
  "editor.formatOnSave": true,
  "editor.codeActionsOnSave": {
    "source.organizeImports": true
  },
  "files.exclude": {
    "**/__pycache__": true,
    "**/*.pyc": true,
    "**/.pytest_cache": true
  }
}
```

---

## Git Workflow

### Gitflow Model

Calibre uses a **simplified Gitflow**:

```
master (main branch)
  ├── feature/add-epub-support
  ├── feature/improve-search
  ├── bugfix/fix-cover-download
  └── release/v6.0.0
```

**React equivalent**: Like feature branches in Next.js development

### Daily Workflow

```bash
# 1. Update master
git checkout master
git pull upstream master

# 2. Create feature branch
git checkout -b feature/my-new-feature

# 3. Make changes
# ... edit files ...

# 4. Stage and commit
git add src/calibre/some_file.py
git commit -m "feat: add new feature"

# 5. Keep branch updated
git fetch upstream
git rebase upstream/master

# 6. Push to your fork
git push origin feature/my-new-feature

# 7. Create pull request on GitHub
```

### Syncing with Upstream

```bash
# Fetch latest changes
git fetch upstream

# Update your master
git checkout master
git merge upstream/master

# Update feature branch (option 1: merge)
git checkout feature/my-feature
git merge upstream/master

# Update feature branch (option 2: rebase - cleaner history)
git checkout feature/my-feature
git rebase upstream/master

# If rebase has conflicts:
# 1. Resolve conflicts in files
# 2. Stage resolved files
git add resolved_file.py
# 3. Continue rebase
git rebase --continue

# Push updated branch (if already pushed)
git push --force-with-lease origin feature/my-feature
```

---

## Branch Management

### Branch Naming Convention

```bash
# Features
feature/add-epub-metadata
feature/improve-search-performance
feature/dark-mode-ui

# Bug fixes
bugfix/fix-cover-download
bugfix/fix-sql-injection
fix/rating-validation

# Documentation
docs/update-api-guide
docs/add-testing-examples

# Refactoring
refactor/simplify-cache-logic
refactor/extract-metadata-parser

# Experiments
experiment/new-conversion-engine
spike/react-ui-prototype
```

### Branch Lifecycle

```bash
# Create branch
git checkout -b feature/my-feature

# Work on branch
git add .
git commit -m "feat: implement feature"

# Push to remote
git push -u origin feature/my-feature

# After PR is merged, delete branch
git checkout master
git pull upstream master
git branch -d feature/my-feature  # Local
git push origin --delete feature/my-feature  # Remote
```

### Long-Running Branches

```bash
# Keep long-running branch updated
git checkout feature/big-refactor

# Regularly merge master
git fetch upstream
git merge upstream/master

# Or rebase (cleaner but more complex)
git rebase upstream/master
```

---

## Commit Guidelines

### Conventional Commits

Calibre uses **Conventional Commits** format:

```
<type>(<scope>): <subject>

<body>

<footer>
```

**Types**:
- `feat`: New feature
- `fix`: Bug fix
- `docs`: Documentation only
- `style`: Code style (formatting, no logic change)
- `refactor`: Code refactoring
- `perf`: Performance improvement
- `test`: Add or update tests
- `build`: Build system changes
- `ci`: CI/CD changes
- `chore`: Other changes (dependencies, etc.)

**Examples**:

```bash
# Feature
git commit -m "feat(search): add fuzzy matching algorithm"

# Bug fix
git commit -m "fix(db): prevent SQL injection in search"

# Documentation
git commit -m "docs(api): add REST endpoint examples"

# Refactoring
git commit -m "refactor(cache): extract metadata refresh logic"

# Breaking change
git commit -m "feat(api)!: change endpoint response format

BREAKING CHANGE: /api/books now returns array instead of object"
```

### Commit Message Guidelines

```bash
# ❌ BAD: Vague, no context
git commit -m "fix bug"
git commit -m "update code"
git commit -m "changes"

# ✅ GOOD: Clear, descriptive
git commit -m "fix(metadata): handle missing ISBN in EPUB files"
git commit -m "feat(conversion): add support for AZW4 format"
git commit -m "perf(db): optimize search query with index"

# ❌ BAD: Too long subject
git commit -m "feat: add new feature that does this and that and also handles edge cases and improves performance"

# ✅ GOOD: Concise subject, detailed body
git commit -m "feat(search): add full-text search support

- Implement PostgreSQL FTS indexing
- Add search highlighting
- Support phrase queries and boolean operators
- Add search performance metrics

Closes #1234"
```

### Atomic Commits

**Each commit should be a single logical change**:

```bash
# ❌ BAD: Multiple unrelated changes
git add src/calibre/db/cache.py  # Database changes
git add src/calibre/gui2/main.py  # GUI changes
git add docs/api.md               # Documentation
git commit -m "fix: various fixes"

# ✅ GOOD: Separate commits
git add src/calibre/db/cache.py
git commit -m "fix(db): prevent race condition in cache update"

git add src/calibre/gui2/main.py
git commit -m "fix(gui): handle missing cover gracefully"

git add docs/api.md
git commit -m "docs(api): add cache API examples"
```

### Interactive Staging

```bash
# Stage parts of files (hunks)
git add -p src/calibre/cache.py

# For each hunk, choose:
# y - stage this hunk
# n - don't stage
# s - split into smaller hunks
# e - manually edit hunk
```

### Amending Commits

```bash
# Amend last commit (not pushed yet)
git add forgotten_file.py
git commit --amend --no-edit

# Amend with new message
git commit --amend -m "fix(db): prevent SQL injection (updated)"

# ⚠️ Never amend pushed commits (unless you're sure)
```

### Interactive Rebase

```bash
# Rewrite last 3 commits
git rebase -i HEAD~3

# In editor:
pick abc123 feat: add feature
pick def456 fix typo
pick ghi789 docs: add example

# Change to:
pick abc123 feat: add feature
fixup def456 fix typo           # Merge into previous
reword ghi789 docs: add example  # Change message

# Save and close editor
```

---

## Code Review Process

### Creating a Pull Request

```bash
# 1. Push your branch
git push origin feature/my-feature

# 2. Go to GitHub
# https://github.com/YOUR_USERNAME/calibre
# Click "Compare & pull request"

# 3. Fill PR template
```

**PR Title**: Use conventional commit format
```
feat(search): add fuzzy matching support
```

**PR Description**:
```markdown
## Summary
Add fuzzy matching to search to improve user experience when searching for books with typos.

## Changes
- Implement Levenshtein distance algorithm
- Add fuzzy matching option to search API
- Update search UI to show fuzzy matches
- Add tests for fuzzy matching

## Testing
- [ ] Unit tests pass
- [ ] Integration tests pass
- [ ] Manual testing with sample library
- [ ] Performance tested with 10,000 books

## Breaking Changes
None

## Related Issues
Closes #1234
Relates to #5678

## Screenshots
(if applicable)
```

### Reviewing Pull Requests

**As a reviewer**:

```bash
# 1. Fetch PR branch
git fetch upstream pull/1234/head:pr-1234
git checkout pr-1234

# 2. Review code
# - Check code quality
# - Run tests
# - Test manually

# 3. Leave feedback on GitHub
# - Approve, Request Changes, or Comment

# 4. If approved, maintainer merges
```

**Review checklist**:

- [ ] **Code Quality**
  - [ ] Follows Python style guide (PEP 8)
  - [ ] No obvious bugs
  - [ ] Good variable/function names
  - [ ] Appropriate comments
  - [ ] No hardcoded values
  - [ ] Error handling present

- [ ] **Testing**
  - [ ] Tests included
  - [ ] Tests pass
  - [ ] Edge cases covered
  - [ ] No flaky tests

- [ ] **Security**
  - [ ] Input validation
  - [ ] No SQL injection
  - [ ] No XSS vulnerabilities
  - [ ] Secure file handling

- [ ] **Performance**
  - [ ] No obvious performance issues
  - [ ] Database queries optimized
  - [ ] Large operations paginated

- [ ] **Documentation**
  - [ ] API changes documented
  - [ ] Inline comments for complex logic
  - [ ] README updated if needed

### Responding to Review Feedback

```bash
# 1. Make requested changes
git checkout feature/my-feature
# ... edit files ...

# 2. Commit changes
git add .
git commit -m "fix: address review feedback"

# 3. Push updates
git push origin feature/my-feature

# 4. Respond to comments on GitHub
# "Fixed in abc123"
# "Good catch! Updated."
```

### Handling Conflicts

```bash
# If PR has conflicts with master:

# 1. Update master
git checkout master
git pull upstream master

# 2. Rebase feature branch
git checkout feature/my-feature
git rebase master

# 3. Resolve conflicts
# Edit conflicting files
git add resolved_file.py
git rebase --continue

# 4. Force push (rebase rewrites history)
git push --force-with-lease origin feature/my-feature
```

---

## Testing Workflow

### Before Committing

```bash
# 1. Run tests
pytest

# 2. Run specific test file
pytest tests/test_cache.py

# 3. Run specific test
pytest tests/test_cache.py::test_metadata_update

# 4. Run with coverage
pytest --cov=calibre --cov-report=html

# 5. Lint code
flake8 src/calibre/

# 6. Format code
black src/calibre/

# 7. Type check
mypy src/calibre/
```

### Continuous Testing

**Use pytest-watch for continuous testing**:

```bash
# Install
pip install pytest-watch

# Run (re-runs tests on file changes)
ptw -- -v

# Watch specific directory
ptw src/calibre/db/ -- tests/test_db/
```

### Pre-commit Hooks

**Automatically run checks before commit**:

`.pre-commit-config.yaml`:
```yaml
repos:
  - repo: https://github.com/psf/black
    rev: 23.1.0
    hooks:
      - id: black
        language_version: python3.10

  - repo: https://github.com/PyCQA/flake8
    rev: 6.0.0
    hooks:
      - id: flake8
        args: ['--max-line-length=120']

  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.4.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-added-large-files
      - id: check-merge-conflict

  - repo: https://github.com/pycqa/isort
    rev: 5.12.0
    hooks:
      - id: isort
        args: ['--profile', 'black']
```

```bash
# Install hooks
pre-commit install

# Run manually
pre-commit run --all-files

# Skip hooks (when necessary)
git commit --no-verify -m "WIP: skip hooks"
```

---

## Continuous Integration

### GitHub Actions

**`.github/workflows/ci.yml`**:

```yaml
name: CI

on:
  push:
    branches: [master]
  pull_request:
    branches: [master]

jobs:
  test:
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os: [ubuntu-latest, macos-latest, windows-latest]
        python-version: ['3.10', '3.11', '3.12']

    steps:
      - uses: actions/checkout@v3

      - name: Set up Python ${{ matrix.python-version }}
        uses: actions/setup-python@v4
        with:
          python-version: ${{ matrix.python-version }}

      - name: Cache pip packages
        uses: actions/cache@v3
        with:
          path: ~/.cache/pip
          key: ${{ runner.os }}-pip-${{ hashFiles('**/requirements*.txt') }}

      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt
          pip install -r requirements-dev.txt

      - name: Lint with flake8
        run: |
          flake8 src/calibre/ --count --select=E9,F63,F7,F82 --show-source --statistics
          flake8 src/calibre/ --count --max-complexity=10 --max-line-length=120 --statistics

      - name: Format check with black
        run: |
          black --check src/calibre/

      - name: Type check with mypy
        run: |
          mypy src/calibre/

      - name: Test with pytest
        run: |
          pytest tests/ -v --cov=calibre --cov-report=xml

      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v3
        with:
          file: ./coverage.xml
          fail_ci_if_error: true

  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Security check with bandit
        run: |
          pip install bandit
          bandit -r src/calibre/ -ll

      - name: Dependency vulnerability check
        run: |
          pip install safety
          safety check
```

### Local CI Simulation

```bash
# Install act (runs GitHub Actions locally)
# https://github.com/nektos/act

# Run CI locally
act

# Run specific job
act -j test

# Run specific event
act pull_request
```

---

## Release Process

### Versioning

Calibre uses **Semantic Versioning** (SemVer):

```
MAJOR.MINOR.PATCH

Example: 6.12.3
- MAJOR: 6 (breaking changes)
- MINOR: 12 (new features, backward compatible)
- PATCH: 3 (bug fixes)
```

**Version bump rules**:
- **MAJOR**: Breaking API changes
- **MINOR**: New features (backward compatible)
- **PATCH**: Bug fixes only

### Creating a Release

```bash
# 1. Update version
# Edit setup.py, __version__.py, etc.
vim src/calibre/__init__.py
# __version__ = '6.13.0'

# 2. Update CHANGELOG
vim CHANGELOG.md
# ## [6.13.0] - 2024-01-15
# ### Added
# - New fuzzy search feature
# ### Fixed
# - SQL injection vulnerability

# 3. Commit version bump
git add .
git commit -m "chore: bump version to 6.13.0"

# 4. Create tag
git tag -a v6.13.0 -m "Release v6.13.0"

# 5. Push tag
git push upstream master --tags

# 6. GitHub Actions builds and releases automatically
```

### Changelog Format

**CHANGELOG.md**:

```markdown
# Changelog

All notable changes to this project will be documented in this file.

## [Unreleased]

### Added
- Fuzzy search support in book search

### Changed
- Improved search performance by 40%

### Deprecated
- Old search API (will be removed in v7.0.0)

### Removed
- None

### Fixed
- SQL injection in search endpoint
- Cover download timeout issues

### Security
- Updated Pillow to fix CVE-2024-1234

## [6.12.0] - 2024-01-01

### Added
- Dark mode UI
- Export library to CSV

### Fixed
- Rating validation bug
```

---

## Documentation Workflow

### Documentation Types

1. **Code Documentation** (Docstrings)
```python
def search_books(query, limit=50):
    """Search books in library.

    Args:
        query (str): Search query using Calibre search syntax
        limit (int): Maximum results to return (default: 50)

    Returns:
        list[int]: Book IDs matching query

    Raises:
        ValueError: If query is invalid

    Example:
        >>> results = search_books('author:orwell tag:fiction')
        >>> print(len(results))
        3
    """
    pass
```

2. **API Documentation** (Markdown)
```markdown
# API Endpoint: GET /api/books

Returns list of books in library.

## Request

```http
GET /api/books?limit=10&offset=0
```

## Response

```json
{
  "books": [...],
  "total": 1234
}
```
```

3. **User Documentation** (Manual)
```markdown
# How to Search Books

To search for books by author:

1. Open the search bar (Ctrl+F)
2. Type `author:orwell`
3. Press Enter
```

### Documentation Review

```bash
# Spell check
aspell check docs/api.md

# Link check
markdown-link-check docs/**/*.md

# Build documentation
cd docs
make html

# Serve locally
python -m http.server 8000
# Visit http://localhost:8000
```

---

## Collaboration Best Practices

### Communication

**Channels**:
- **GitHub Issues**: Bug reports, feature requests
- **Pull Requests**: Code review, discussion
- **Discussions**: General questions, ideas
- **Email**: Private matters

**Best practices**:
```markdown
# ✅ GOOD issue report

## Bug Report: Search crashes with special characters

**Environment**:
- Calibre version: 6.12.0
- OS: Ubuntu 22.04
- Python: 3.10.8

**Steps to reproduce**:
1. Open search bar
2. Type: `author:"O'Brien"`
3. Press Enter

**Expected**: Show books by O'Brien
**Actual**: Application crashes

**Error log**:
```
Traceback (most recent call last):
  File "search.py", line 123
    ...
```

**Additional context**:
Works fine without apostrophe: `author:OBrien`
```

### Code Ownership

**CODEOWNERS file**:
```
# Database layer
/src/calibre/db/ @database-team

# GUI
/src/calibre/gui2/ @gui-team

# Conversion
/src/calibre/ebooks/conversion/ @conversion-team

# Documentation
/docs/ @docs-team
```

### Pair Programming

```bash
# Use VS Code Live Share
# 1. Install Live Share extension
# 2. Click "Live Share" in status bar
# 3. Share link with collaborator

# Or use tmux for terminal sharing
tmux new-session -s pairing
# Collaborator: tmux attach-session -t pairing
```

### Knowledge Sharing

**Weekly sync**:
- What did you work on?
- Any blockers?
- What's next?

**Documentation**:
- Update docs when code changes
- Write ADRs (Architecture Decision Records)
- Keep README updated

**Code reviews**:
- Review others' PRs promptly
- Learn from reviews on your PRs
- Share interesting findings

---

## Quick Reference

### Common Commands

```bash
# Setup
git clone <repo>
cd repo
python -m venv venv
source venv/bin/activate
pip install -e .

# Daily workflow
git checkout master
git pull upstream master
git checkout -b feature/my-feature
# ... make changes ...
git add .
git commit -m "feat: add feature"
git push origin feature/my-feature

# Testing
pytest
pytest --cov
flake8 src/
black src/

# Keep branch updated
git fetch upstream
git rebase upstream/master

# Clean up
git checkout master
git branch -d feature/my-feature
```

### Git Aliases

Add to `~/.gitconfig`:

```ini
[alias]
    st = status
    co = checkout
    br = branch
    ci = commit
    unstage = reset HEAD --
    last = log -1 HEAD
    visual = log --oneline --graph --all --decorate

    # Update from upstream
    sync = !git fetch upstream && git rebase upstream/master

    # Clean merged branches
    cleanup = !git branch --merged | grep -v '\\*' | xargs -n 1 git branch -d

    # Amend without editing message
    amend = commit --amend --no-edit
```

---

## Next Steps

Now that you understand the workflow:

1. **Practice**: Make your first contribution!
2. **Read**: [TESTING_GUIDE.md](./TESTING_GUIDE.md) - Write tests for your code
3. **Read**: [CODE_REVIEW_GUIDE.md](./CODE_REVIEW_GUIDE.md) - Learn review techniques
4. **Explore**: Check out [good first issue](https://github.com/kovidgoyal/calibre/labels/good%20first%20issue) labels

---

## Additional Resources

### Tools
- [GitHub CLI](https://cli.github.com/) - Manage PRs from terminal
- [act](https://github.com/nektos/act) - Run GitHub Actions locally
- [pre-commit](https://pre-commit.com/) - Git hook manager
- [commitizen](https://github.com/commitizen/cz-cli) - Commit message helper

### Guides
- [Conventional Commits](https://www.conventionalcommits.org/)
- [Semantic Versioning](https://semver.org/)
- [Keep a Changelog](https://keepachangelog.com/)
- [GitHub Flow](https://guides.github.com/introduction/flow/)

---

**Remember**: Good workflow is about consistency, communication, and continuous improvement. The tools and processes exist to help the team work together effectively!
