# GIT & VERSION CONTROL
## Exam-Style Study Notes

---

## TOPIC: Git Mastery for Professional Development

### WHY? (Problem Statement)

**Without Version Control:**
- No history of changes
- Can't collaborate without overwriting
- No rollback when things break
- No code review process
- "It works on my machine" syndrome

**What Git Solves:**
| Problem | Git Solution |
|---------|--------------|
| Change tracking | Immutable commit history |
| Collaboration | Branches, merges, remotes |
| Rollback | `git reset`, `git revert` |
| Code review | Pull/Merge Requests |
| CI/CD integration | Hooks, tags, release branches |
| Experimentation | Feature branches, stashing |

**Real-World Analogy:**
- Git = Time machine + Collaboration platform
- Commits = Save points in a game
- Branches = Parallel universes
- Merge = Merging timelines

---

### HOW? (Internal Mechanism)

#### 1. **Git Object Model**

```
┌─────────────────────────────────────────────────────────────┐
│                    .git/objects/                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  BLOB (file content)                                       │
│  └── SHA-1 hash of content                                 │
│       │                                                    │
│       ▼                                                    │
│  TREE (directory snapshot)                                 │
│  └── References blobs + other trees                        │
│       │                                                    │
│       ▼                                                    │
│  COMMIT (snapshot + metadata)                              │
│  ├── Tree (root directory)                                 │
│  ├── Parent commit(s)                                      │
│  ├── Author/Committer                                      │
│  ├── Timestamp                                             │
│  └── Message                                               │
│       │                                                    │
│       ▼                                                    │
│  REF (branch/tag) → Commit SHA                             │
│                                                             │
└─────────────────────────────────────────────────────────────┘

Key Insight: Git stores SNAPSHOTS, not diffs!
Each commit = complete filesystem snapshot (deduplicated via SHA-1)
```

#### 2. **Three Trees / Three States**

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  WORKING    │     │   STAGING   │     │   REPO      │
│  DIRECTORY  │────►│   AREA      │────►│  (COMMITS)  │
│  (untracked │     │  (index)    │     │  (history)  │
│   changes)  │     │             │     │             │
└─────────────┘     └─────────────┘     └─────────────┘
      │                   │                   │
  git add            git commit            git push
  git rm             git reset             git fetch
  git checkout       git restore           git pull
```

#### 3. **Branching = Moving Pointers**

```
main:     A ── B ── C ── D ── E
              \           \
feature:        F ── G ── H ── I
                    \           \
hotfix:              J ── K ── L ── M

# HEAD points to current branch
# Branch = lightweight pointer to commit
# Fast-forward = move pointer forward (no merge commit)
# Merge commit = new commit with two parents
```

---

### WHAT? (Key Commands & Workflows)

#### 1. **Essential Daily Commands**

```bash
# REPOSITORY SETUP
git init                          # New repo
git clone <url>                   # Clone existing
git clone --depth 1 <url>         # Shallow clone (CI)

# CONFIGURATION
git config --global user.name "Name"
git config --global user.email "email@domain.com"
git config --global init.defaultBranch main
git config --global pull.rebase false
git config --global push.autoSetupRemote true

# STATUS & INSPECTION
git status                        # Working tree status
git status -s                     # Short format
git diff                          # Unstaged changes
git diff --staged                 # Staged changes
git diff HEAD                     # All changes
git log --oneline -10             # Recent commits
git log --graph --all --oneline   # Visual history
git show <commit>                 # Commit details

# STAGING
git add <file>                    # Stage file
git add -p                        # Interactive patch staging
git add -A                        # Stage all (new, modified, deleted)
git reset HEAD <file>             # Unstage file

# COMMITTING
git commit -m "message"           # Commit staged
git commit -am "message"          # Stage tracked + commit
git commit --amend                # Modify last commit
git commit --amend --no-edit      # Keep message, add staged

# BRANCHING
git branch                        # List local branches
git branch -a                     # All branches (local + remote)
git branch <name>                 # Create branch
git branch -d <name>              # Delete (merged)
git branch -D <name>              # Force delete
git switch <name>                 # Switch branch (Git 2.23+)
git switch -c <name>              # Create + switch
git checkout -b <name>            # Legacy create+switch

# MERGING & REBASING
git merge <branch>                # Merge into current
git merge --no-ff <branch>        # Always create merge commit
git rebase <branch>               # Replay commits on top
git rebase -i HEAD~3              # Interactive rebase (last 3)
git rebase --abort                # Cancel rebase
git rebase --continue             # Continue after conflict resolution

# REMOTE
git remote -v                     # List remotes
git remote add origin <url>       # Add remote
git fetch                         # Download objects/refs
git pull                          # Fetch + merge
git pull --rebase                 # Fetch + rebase
git push                          # Push current branch
git push -u origin <branch>       # Push + set upstream
git push --force-with-lease       # Safe force push

# TAGGING
git tag                           # List tags
git tag v1.0.0                    # Lightweight tag
git tag -a v1.0.0 -m "Release"    # Annotated tag
git push origin v1.0.0            # Push tag
git push origin --tags            # Push all tags
```

#### 2. **Advanced Workflows**

```bash
# INTERACTIVE REBASE (Clean history before PR)
git rebase -i HEAD~5
# Commands: pick, reword, edit, squash, fixup, drop, exec

# CHERRY-PICK (Apply specific commit)
git cherry-pick <commit-sha>
git cherry-pick --no-commit <sha>  # Stage only

# REFLLOG (Recovery!)
git reflog                         # All HEAD movements
git reset --hard HEAD@{3}          # Go back 3 steps

# STASHING (Save work in progress)
git stash                          # Save changes
git stash list                     # List stashes
git stash pop                      # Apply + remove latest
git stash apply                    # Apply without removing
git stash drop                     # Delete stash
git stash push -m "message"        # Named stash

# BISECT (Find bug commit)
git bisect start
git bisect bad                     # Current is bad
git bisect good v1.0.0             # Known good
# Git checks out middle commit
# Test, then: git bisect good/bad
git bisect reset                   # End session

# SUBMODULES
git submodule add <url> path
git submodule update --init --recursive
git submodule foreach git pull origin main

# WORKTREES (Multiple working dirs)
git worktree add ../hotfix hotfix-branch
git worktree list
git worktree remove ../hotfix
```

#### 3. **Conflict Resolution**

```bash
# During merge/rebase/cherry-pick conflict:
# 1. Edit conflicted files (<<<<<<< HEAD ... ======= ... >>>>>>> branch)
# 2. Stage resolved files
git add <resolved-file>
# 3. Continue
git merge --continue        # or git rebase --continue

# Abort
git merge --abort
git rebase --abort

# Tools
git mergetool               # Launch configured tool (vimdiff, meld, etc.)
git diff --check            # Whitespace errors
```

#### 4. **Git Aliases (Productivity)**

```bash
# ~/.gitconfig
[alias]
  # Status
  st = status -s
  ss = status
  
  # Log
  lg = log --graph --pretty=format:'%C(yellow)%h%Creset %C(cyan)%ad%Creset %C(green)%an%Creset %s %C(red)%d%Creset' --date=short
  lol = log --oneline --graph --decorate --all
  
  # Diff
  d = diff
  ds = diff --staged
  dw = diff --word-diff
  
  # Branch
  co = checkout
  cob = checkout -b
  br = branch
  bra = branch -a
  
  # Commit
  cm = commit -m
  cam = commit -am
  amend = commit --amend --no-edit
  
  # Push/Pull
  pu = push
  pur = push --force-with-lease
  pl = pull
  plr = pull --rebase
  
  # Stash
  sta = stash
  sta = stash push -m
  stl = stash list
  stp = stash pop
  
  # Reset
  rh = reset --hard
  r1 = reset HEAD~1
  r2 = reset HEAD~2
  
  # Cleanup
  clean = clean -fd
  
  # Recovery
  reflog = reflog --oneline
```

---

### PATTERNS & ANTI-PATTERNS

| Pattern | Anti-Pattern | Why Avoid |
|---------|--------------|-----------|
| **Small, focused commits** | One giant "WIP" commit | Bisect, review, revert impossible |
| **Descriptive messages** | "fix", "update", "changes" | No context for history |
| **Feature branches** | Direct commits to main | No review, broken main |
| **Rebase before PR** | Merge main into feature | Clean history, easier review |
| **Conventional Commits** | Random message format | Auto-changelog, semantic release |
| **Force-with-lease** | `push -f` | Protects others' work |
| **Signed commits** | Unverified authorship | Security, compliance |

#### Conventional Commits (Standard)
```
<type>[scope]: <description>

[body]

[footer]

Types: feat, fix, docs, style, refactor, perf, test, chore, build, ci, revert

Examples:
feat(auth): add OAuth2 login
fix(api): handle null response in user endpoint
refactor(utils): extract date formatting
docs(readme): update installation steps
chore(deps): upgrade typescript to 5.3
```

---

### INTERVIEW QUESTIONS (Top 15)

#### 1. **Conceptual**: "Git merge vs rebase - when to use which?"

**Answer:**
```
MERGE:
✓ Preserves exact history
✓ Non-destructive
✓ Shows branch topology
✗ Creates merge commits (clutter)
✗ Harder to read linear history

REBASE:
✓ Linear, clean history
✓ Easier bisect/debugging
✓ No merge commits
✗ Rewrites history (DANGEROUS on shared branches)
✗ Loses context of when branch diverged

RULE:
- Local/private branches: REBASE freely
- Shared/public branches: NEVER REBASE (use merge)
- Before PR: rebase onto target branch
- Integration: merge (or squash merge)
```

#### 2. **Code**: "Undo last commit but keep changes"

```bash
# Soft reset - keep staged
git reset --soft HEAD~1

# Mixed reset (default) - keep unstaged
git reset HEAD~1

# Hard reset - DISCARD changes (DANGEROUS!)
git reset --hard HEAD~1

# Revert (safe, creates new commit)
git revert HEAD
```

#### 3. **Design**: "Team workflow for 10 developers"

```bash
# TRUNK-BASED DEVELOPMENT (Recommended)
main ──► ──► ──► ──► ──► ──►
   │      │      │      │
   │    feat    fix   feat
   │    ┌───┐  ┌───┐  ┌───┐
   └────┤PR├──┤PR├──┤PR├──►
        └───┘  └───┘  └───┘
        
Rules:
1. Main always deployable
2. Short-lived branches (< 1 day)
3. PR required, 1+ approval
4. CI passes before merge
5. Squash merge (clean history)
6. Delete branch after merge
```

#### 4. **Code**: "Find commit that introduced bug"

```bash
# Git bisect (binary search)
git bisect start
git bisect bad                    # Current broken
git bisect good v2.1.0            # Last known good
# Git checks out middle commit
# Run tests
git bisect good                   # If passes
git bisect bad                    # If fails
# Repeat until found
git bisect reset                  # Cleanup

# Automation
git bisect run npm test           # Automated!
```

#### 5. **Debugging**: "Accidentally committed secrets - remove from history"

```bash
# BFG Repo-Cleaner (fast, recommended)
java -jar bfg.jar --delete-files id_rsa .git
java -jar bfg.jar --replace-text passwords.txt .git

# git filter-repo (modern replacement for filter-branch)
git filter-repo --path .env --invert-paths

# Manual (slow, for small repos)
git filter-branch --force --index-filter \
  'git rm --cached --ignore-unmatch .env' \
  --prune-empty --tag-name-filter cat -- --all

# AFTER: Force push ALL branches + tags
git push origin --force --all
git push origin --force --tags

# CRITICAL: Rotate the secret immediately!
```

#### 6. **Advanced**: "Explain `git push --force-with-lease` vs `--force`"

```bash
# --force: Overwrites remote, DESTROYS others' work
git push --force

# --force-with-lease: Only pushes if remote matches your remote-tracking branch
# Fails if someone else pushed since your last fetch
git push --force-with-lease

# Scenario:
# You: fetch → commit → push --force-with-lease ✓
# Teammate: push → You: push --force-with-lease ✗ (fails, protects their work)
```

#### 7. **Configuration**: "Useful .gitconfig settings"

```ini
[core]
  editor = code --wait
  autocrlf = input
  pager = delta
  hooksPath = ~/.githooks

[init]
  defaultBranch = main

[pull]
  rebase = false
  ff = only

[push]
  autoSetupRemote = true
  followTags = true

[merge]
  conflictStyle = diff3
  tool = vscode

[diff]
  tool = vscode
  algorithm = histogram

[rebase]
  autoStash = true
  updateRefs = true

[fetch]
  prune = true

[log]
  date = short

[alias]
  # (see aliases section above)

[commit]
  gpgsign = true

[tag]
  gpgsign = true

[gpg]
  program = gpg2
```

#### 8. **Recovery**: "Deleted a branch - how to recover?"

```bash
# 1. Reflog (local)
git reflog
git checkout -b recovered-branch <sha>

# 2. If pushed to remote
git fetch origin
git branch -a | grep deleted-branch
git checkout -b recovered origin/deleted-branch

# 3. GitHub/GitLab UI: "Restore branch" button (within 30 days)

# 4. If force-pushed and lost locally
# Check reflog on another clone, or GitHub Events API
```

---

### GOTCHAS & EDGE CASES

- [ ] **`git pull` = fetch + merge** (creates merge commit). Use `pull --rebase` for linear
- [ ] **`git checkout`** overloaded (switch branch OR restore file). Use `switch`/`restore`
- [ ] **Empty directories** not tracked. Add `.gitkeep` file
- [ ] **Case-sensitive filenames** on Mac/Windows. `git config core.ignorecase false`
- [ ] **Line endings**: `core.autocrlf=input` (LF in repo, CRLF on Windows checkout)
- [ ] **Submodules** point to commit, not branch. Update with `git submodule update --remote`
- [ ] **`git add .` vs `git add -A`**: `.` = current dir, `-A` = entire repo
- [ ] **Shallow clone** can't push. `git fetch --unshallow` to fix
- [ ] **Detached HEAD**: You're on a commit, not branch. Create branch to save: `git switch -c new-branch`

---

### CODE SNIPPETS TO MEMORIZE

#### 1. **Complete Feature Workflow**
```bash
# Start feature
git switch -c feat/user-dashboard main

# Work, commit often
git add -p
git commit -m "feat(dashboard): add user stats widget"

# Keep updated with main
git fetch origin
git rebase origin/main

# Push for review
git push -u origin feat/user-dashboard

# After PR approved + merged
git switch main
git pull
git branch -d feat/user-dashboard
git push origin --delete feat/user-dashboard
```

#### 2. **Hotfix Workflow**
```bash
# From main
git switch -c hotfix/security-patch main

# Fix, test, commit
git commit -am "fix(security): sanitize user input"

# Push
git push -u origin hotfix/security-patch

# PR → Merge → Deploy

# Backport to develop if needed
git switch develop
git cherry-pix <hotfix-commit>
```

#### 3. **Release Workflow**
```bash
# Create release branch
git switch -c release/v2.0.0 main

# Version bump, changelog
npm version minor  # or manual
git commit -am "chore(release): v2.0.0"

# Tag
git tag -a v2.0.0 -m "Release v2.0.0"

# Merge to main + develop
git switch main
git merge --no-ff release/v2.0.0
git push origin main --tags

git switch develop
git merge --no-ff release/v2.0.0
git push origin develop

# Cleanup
git branch -d release/v2.0.0
```

#### 4. **Git Hooks (Husky + lint-staged)**
```json
// package.json
{
  "husky": {
    "hooks": {
      "pre-commit": "lint-staged",
      "commit-msg": "commitlint -E HUSKY_GIT_PARAMS"
    }
  },
  "lint-staged": {
    "*.{js,ts,tsx}": ["eslint --fix", "prettier --write"],
    "*.{json,md,css}": ["prettier --write"]
  }
}
```

---

### PRACTICE PROBLEMS

1. **Create** a repo with: main, develop, feature/* branches. Practice PR workflow
2. **Simulate** conflict: two branches modify same line. Resolve with merge/rebase
3. **Practice** interactive rebase: squash 5 commits into 1, reorder, edit messages
4. **Debug** with bisect: introduce bug in history, find it automatically
5. **Recover** deleted branch using reflog and remote

---

### ANKI FLASHCARDS

| Front | Back |
|-------|------|
| Git stores snapshots or diffs? | Snapshots (complete filesystem state per commit) |
| Merge vs Rebase | Merge: preserves history, creates merge commit. Rebase: linear, rewrites history |
| When NEVER to rebase | On shared/public branches (others may have based work on it) |
| `git pull` = ? | `git fetch` + `git merge` |
| `push --force` vs `--force-with-lease` | Force: overwrites. Force-with-lease: fails if remote has new commits |
| Reflog purpose | Local history of HEAD movements (recovery!) |
| Bisect algorithm | Binary search through commit history |
| Detached HEAD | HEAD points to commit, not branch. Create branch to save |
| Conventional commit format | `type(scope): description` - enables auto-changelog |
| Force-with-lease protects | Teammate's work pushed after your last fetch |

---

## NEXT TOPIC: `08-BUILD-TOOLS.md`

> **Study Tip**: Set up a repo with Husky, Commitlint, Conventional Commits. Practice the full feature → PR → merge → release workflow. Use `git rebase -i` daily to clean commits before pushing.