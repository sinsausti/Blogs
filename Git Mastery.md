# Git Mastery: From Zero to Hero Developer Guide

Hey developers! 👋

If you've ever lost code, struggled with team collaboration, or wondered "what changed and when?", then Git is about to revolutionize your development workflow. This version control system has become the backbone of modern software development, and mastering it is essential for any serious developer.

Let's dive deep into Git – from the fundamentals to advanced techniques that'll make you a version control wizard!

## Why Git Matters

Before we jump into commands, let's understand why Git has conquered the development world:

- **Distributed version control**: Every clone is a full backup
- **Branching made easy**: Experiment without fear
- **Collaboration superpowers**: Multiple developers, zero conflicts (well, mostly!)
- **Time travel**: Go back to any point in your project's history
- **Industry standard**: Used by virtually every tech company

## Part 1: Git Fundamentals - Your First Steps

### Installation and Initial Setup

**Install Git:**
```bash
# Ubuntu/Debian
sudo apt install git

# macOS
brew install git

# Windows
# Download from https://git-scm.com/
```

**Essential Configuration:**
```bash
# Set your identity (required for commits)
git config --global user.name "Your Name"
git config --global user.email "your.email@example.com"

# Set default branch name
git config --global init.defaultBranch main

# Useful aliases
git config --global alias.st status
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.cm commit
git config --global alias.lg "log --oneline --graph --decorate --all"

# Better diff and merge tools
git config --global merge.tool vimdiff
git config --global diff.tool vimdiff
```

### Understanding Git's Three Trees

Git works with three main areas:

1. **Working Directory**: Your actual files
2. **Staging Area (Index)**: Files ready to be committed
3. **Repository**: Your committed history

```bash
# Check what's happening in all three areas
git status

# See differences between working directory and staging
git diff

# See differences between staging and last commit
git diff --cached

# See differences between working directory and last commit
git diff HEAD
```

### Basic Git Workflow

#### Starting a New Project

```bash
# Initialize a new repository
mkdir my-awesome-project
cd my-awesome-project
git init

# Create your first file
echo "# My Awesome Project" > README.md

# Stage the file
git add README.md

# Make your first commit
git commit -m "Initial commit: Add README"
```

#### The Daily Git Cycle

```bash
# Check current status
git status

# Add files to staging area
git add filename.txt          # Add specific file
git add .                     # Add all files in current directory
git add *.js                  # Add all JavaScript files
git add -A                    # Add all changes (including deletions)

# Commit changes
git commit -m "Add user authentication feature"

# View commit history
git log
git log --oneline            # Condensed view
git log --graph --all        # Visual branch representation
```

## Part 2: Branching - Git's Superpower

Branching is where Git truly shines. It's fast, lightweight, and encourages experimentation.

### Basic Branch Operations

```bash
# Create a new branch
git branch feature/user-login

# Switch to the branch
git checkout feature/user-login

# Create and switch in one command
git checkout -b feature/user-dashboard

# List all branches
git branch                   # Local branches
git branch -r               # Remote branches
git branch -a               # All branches

# Delete a branch
git branch -d feature/old-feature    # Safe delete (merged only)
git branch -D feature/old-feature    # Force delete
```

### Modern Git: Using `git switch` and `git restore`

Git 2.23+ introduced cleaner commands:

```bash
# Switch branches (replaces checkout for branch switching)
git switch main
git switch -c feature/new-feature    # Create and switch

# Restore files (replaces checkout for file restoration)
git restore filename.txt             # Restore from staging
git restore --staged filename.txt    # Unstage file
git restore --source=HEAD~1 filename.txt  # Restore from specific commit
```

### Branch Management Strategies

#### Feature Branch Workflow

```bash
# Start new feature
git switch main
git pull origin main
git switch -c feature/payment-integration

# Work on feature
echo "Payment code here" > payment.js
git add payment.js
git commit -m "Add payment integration"

# Push feature branch
git push -u origin feature/payment-integration

# Merge back to main (after code review)
git switch main
git merge feature/payment-integration
git branch -d feature/payment-integration
```

#### Git Flow Model

```bash
# Main branches
main          # Production-ready code
develop       # Integration branch

# Supporting branches
feature/*     # New features
release/*     # Release preparation
hotfix/*      # Emergency fixes

# Example: Starting a new feature
git switch develop
git switch -c feature/shopping-cart

# Example: Creating a release
git switch develop
git switch -c release/v1.2.0

# Example: Emergency hotfix
git switch main
git switch -c hotfix/critical-security-fix
```

## Part 3: Remote Repositories - Collaboration Magic

### Working with Remotes

```bash
# Add remote repository
git remote add origin https://github.com/username/repo.git

# View remotes
git remote -v

# Push to remote
git push origin main
git push -u origin feature/new-feature  # Set upstream tracking

# Pull from remote
git pull origin main                     # Fetch + merge
git fetch origin                         # Fetch only
git merge origin/main                    # Merge after fetch

# Clone existing repository
git clone https://github.com/username/repo.git
git clone https://github.com/username/repo.git my-local-name
```

### Collaboration Workflows

#### Centralized Workflow

```bash
# Everyone works on main branch
git clone https://github.com/team/project.git
cd project

# Make changes
git add .
git commit -m "Add new feature"

# Always pull before pushing
git pull origin main
git push origin main
```

#### Feature Branch Workflow

```bash
# Create feature branch
git switch -c feature/user-profile

# Push feature branch
git push -u origin feature/user-profile

# Create Pull Request (via GitHub/GitLab web interface)
# After review and approval, merge via web interface
```

#### Forking Workflow

```bash
# Fork repository on GitHub, then clone your fork
git clone https://github.com/yourusername/upstream-repo.git
cd upstream-repo

# Add upstream remote
git remote add upstream https://github.com/original-owner/upstream-repo.git

# Keep your fork updated
git fetch upstream
git switch main
git merge upstream/main
git push origin main
```

## Part 4: Advanced Git Techniques

### Rewriting History (Use with Caution!)

#### Interactive Rebase

```bash
# Rebase last 3 commits interactively
git rebase -i HEAD~3

# In the interactive editor, you can:
# pick = use commit as-is
# reword = change commit message
# edit = pause to amend commit
# squash = combine with previous commit
# fixup = like squash but discard message
# drop = remove commit
```

#### Amending Commits

```bash
# Change the last commit message
git commit --amend -m "Better commit message"

# Add files to the last commit
git add forgotten-file.txt
git commit --amend --no-edit

# Amend author information
git commit --amend --author="New Author <email@example.com>"
```

#### Cherry-picking

```bash
# Apply specific commit to current branch
git cherry-pick abc123

# Cherry-pick multiple commits
git cherry-pick abc123 def456 ghi789

# Cherry-pick without committing (for modifications)
git cherry-pick --no-commit abc123
```

### Stashing - Temporary Storage

```bash
# Stash current changes
git stash
git stash save "Work in progress on feature X"

# List stashes
git stash list

# Apply stash
git stash apply            # Apply most recent
git stash apply stash@{2}  # Apply specific stash

# Pop stash (apply and remove)
git stash pop

# Create branch from stash
git stash branch feature/from-stash stash@{1}

# Clear all stashes
git stash clear
```

### Handling Merge Conflicts

```bash
# When merge conflicts occur
git merge feature/conflicting-branch

# Git will mark conflicted files
# Edit files to resolve conflicts (look for <<<<<<< and >>>>>>>)

# After resolving conflicts
git add resolved-file.txt
git commit

# Abort merge if needed
git merge --abort
```

**Conflict Resolution Example:**
```bash
<<<<<<< HEAD
function calculateTotal(price) {
    return price * 1.1; // 10% tax
}
=======
function calculateTotal(price, tax = 0.08) {
    return price * (1 + tax);
}
>>>>>>> feature/tax-calculation
```

Resolve to:
```bash
function calculateTotal(price, tax = 0.1) {
    return price * (1 + tax); // Default 10% tax
}
```

### Reset vs Revert

```bash
# Reset (changes history - dangerous on shared branches)
git reset --soft HEAD~1    # Keep changes staged
git reset --mixed HEAD~1   # Keep changes in working directory
git reset --hard HEAD~1    # Discard all changes

# Revert (creates new commit - safe for shared branches)
git revert HEAD            # Revert last commit
git revert abc123          # Revert specific commit
```

## Part 5: Git Best Practices and Workflows

### Commit Best Practices

#### Writing Good Commit Messages

```bash
# Good commit message structure:
# <type>(<scope>): <subject>
#
# <body>
#
# <footer>

# Examples:
git commit -m "feat(auth): add user login functionality"
git commit -m "fix(ui): resolve button alignment issue in Safari"
git commit -m "docs: update API documentation for v2.0"
git commit -m "refactor(database): optimize user query performance"
```

#### Conventional Commits

```bash
feat:     # New feature
fix:      # Bug fix
docs:     # Documentation changes
style:    # Formatting changes
refactor: # Code refactoring
test:     # Adding tests
chore:    # Maintenance tasks

# Examples:
git commit -m "feat: add user dashboard"
git commit -m "fix: resolve memory leak in image processing"
git commit -m "docs: add installation instructions"
```

### Advanced Workflows

#### GitFlow in Practice

```bash
# Initialize GitFlow
git flow init

# Start new feature
git flow feature start payment-system

# Finish feature
git flow feature finish payment-system

# Start release
git flow release start v1.2.0

# Finish release
git flow release finish v1.2.0

# Emergency hotfix
git flow hotfix start critical-bug
git flow hotfix finish critical-bug
```

#### GitHub Flow (Simplified)

```bash
# 1. Create branch from main
git switch main
git pull origin main
git switch -c feature/new-awesome-feature

# 2. Make commits
git add .
git commit -m "Add awesome feature"

# 3. Push and create Pull Request
git push -u origin feature/new-awesome-feature

# 4. Deploy and test (via CI/CD)
# 5. Merge to main (via GitHub interface)
# 6. Delete feature branch
```

### Useful Git Aliases and Scripts

Add these to your `.gitconfig`:

```bash
[alias]
    # Shortcuts
    st = status
    co = checkout
    sw = switch
    br = branch
    cm = commit
    
    # Useful combinations
    ac = !git add -A && git commit
    save = !git add -A && git commit -m 'SAVEPOINT'
    wip = !git add -u && git commit -m "WIP"
    undo = reset HEAD~1 --mixed
    amend = commit -a --amend
    
    # Logging
    lg = log --color --graph --pretty=format:'%Cred%h%Creset -%C(yellow)%d%Creset %s %Cgreen(%cr) %C(bold blue)<%an>%Creset' --abbrev-commit
    hist = log --pretty=format:'%h %ad | %s%d [%an]' --graph --date=short
    
    # Cleanup
    cleanup = "!git branch --merged | grep -v '\\*\\|main\\|develop' | xargs -n 1 git branch -d"
```

## Part 6: Git Hooks and Automation

### Client-side Hooks

```bash
# Pre-commit hook (runs before commit)
# .git/hooks/pre-commit
#!/bin/sh
# Run tests before commit
npm test
if [ $? -ne 0 ]; then
    echo "Tests must pass before commit!"
    exit 1
fi
```

```bash
# Pre-push hook (runs before push)
# .git/hooks/pre-push
#!/bin/sh
# Prevent pushing to main branch
protected_branch='main'
current_branch=$(git symbolic-ref HEAD | sed -e 's,.*/\(.*\),\1,')

if [ $protected_branch = $current_branch ]; then
    echo "Direct push to main branch is not allowed"
    exit 1
fi
```

### Server-side Hooks

```bash
# Post-receive hook (runs after receiving push)
# Can trigger deployments, notifications, etc.
#!/bin/sh
# Deploy to staging when develop branch is pushed
while read oldrev newrev refname; do
    branch=$(git rev-parse --symbolic --abbrev-ref $refname)
    if [ "develop" = "$branch" ]; then
        echo "Deploying to staging..."
        # Deployment script here
    fi
done
```

## Part 7: Troubleshooting and Recovery

### Common Git Problems and Solutions

#### "I committed to the wrong branch"

```bash
# Move commits to correct branch
git switch correct-branch
git cherry-pick wrong-branch~2..wrong-branch

# Remove commits from wrong branch
git switch wrong-branch
git reset --hard HEAD~2
```

#### "I need to undo my last commit"

```bash
# Keep changes in working directory
git reset --soft HEAD~1

# Discard changes completely
git reset --hard HEAD~1

# Create revert commit (safe for shared repos)
git revert HEAD
```

#### "I accidentally deleted a file"

```bash
# Restore from last commit
git restore deleted-file.txt

# Restore from specific commit
git restore --source=HEAD~3 deleted-file.txt

# Find deleted file in history
git log --all --full-history -- deleted-file.txt
```

#### "My repository is corrupted"

```bash
# Check repository integrity
git fsck

# Garbage collection and cleanup
git gc --aggressive --prune=now

# Recover from backup (if you have one)
git clone /path/to/backup.git recovered-repo
```

### Git Reflog - Your Safety Net

```bash
# View reflog (shows all HEAD movements)
git reflog

# Recover "lost" commits
git reflog
# Find the commit hash you want to recover
git switch -c recovery-branch abc1234
```

## Part 8: Advanced Git Features

### Submodules - Managing Dependencies

```bash
# Add submodule
git submodule add https://github.com/user/library.git lib/library

# Clone repository with submodules
git clone --recursive https://github.com/user/main-project.git

# Update submodules
git submodule update --remote

# Remove submodule
git submodule deinit lib/library
git rm lib/library
```

### Git Worktrees - Multiple Working Directories

```bash
# Create new worktree
git worktree add ../feature-branch feature/new-feature

# List worktrees
git worktree list

# Remove worktree
git worktree remove ../feature-branch
```

### Bisect - Finding Bugs

```bash
# Start bisect session
git bisect start
git bisect bad HEAD          # Current commit is bad
git bisect good abc123       # This commit was good

# Git will checkout middle commit
# Test and mark as good or bad
git bisect good              # if test passes
git bisect bad               # if test fails

# Continue until bug is found
git bisect reset             # End session
```

### Advanced Merging Strategies

```bash
# Merge strategies
git merge --no-ff feature-branch     # Always create merge commit
git merge --squash feature-branch    # Squash all commits into one
git merge -X theirs feature-branch   # Prefer their changes in conflicts

# Merge specific files only
git checkout feature-branch -- specific-file.txt
git commit -m "Merge specific file from feature branch"
```

## Part 9: Git Performance and Optimization

### Large Repository Management

```bash
# Shallow clone (limited history)
git clone --depth 1 https://github.com/user/large-repo.git

# Partial clone (limited object types)
git clone --filter=blob:none https://github.com/user/repo.git

# Clean up repository
git gc --aggressive
git prune --expire=now
git repack -Ad
```

### Git LFS (Large File Storage)

```bash
# Install Git LFS
git lfs install

# Track large files
git lfs track "*.psd"
git lfs track "*.zip"

# Add .gitattributes to repository
git add .gitattributes

# Normal git workflow for LFS files
git add large-file.psd
git commit -m "Add design files"
git push origin main
```

## Part 10: CI/CD Integration

### GitHub Actions Example

```yaml
# .github/workflows/ci.yml
name: CI/CD Pipeline
on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    - name: Setup Node.js
      uses: actions/setup-node@v2
      with:
        node-version: '16'
    - run: npm install
    - run: npm test
    
  deploy:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
    - uses: actions/checkout@v2
    - name: Deploy to production
      run: |
        echo "Deploying to production..."
        # Deployment commands here
```

### GitLab CI Example

```yaml
# .gitlab-ci.yml
stages:
  - test
  - deploy

test:
  stage: test
  script:
    - npm install
    - npm test
  only:
    - merge_requests
    - main

deploy:
  stage: deploy
  script:
    - echo "Deploying to production..."
    # Deployment commands
  only:
    - main
```

## Git Command Cheat Sheet

### Essential Commands

```bash
# Repository Setup
git init                              # Initialize repository
git clone <url>                       # Clone repository
git remote add origin <url>           # Add remote

# Basic Workflow
git status                            # Check status
git add <file>                        # Stage file
git add .                             # Stage all files
git commit -m "message"               # Commit with message
git push origin <branch>              # Push to remote
git pull origin <branch>              # Pull from remote

# Branching
git branch                            # List branches
git branch <name>                     # Create branch
git switch <branch>                   # Switch branch
git switch -c <branch>                # Create and switch
git merge <branch>                    # Merge branch
git branch -d <branch>                # Delete branch

# History and Inspection
git log                               # Commit history
git log --oneline                     # Condensed history
git show <commit>                     # Show commit details
git diff                              # Show differences
git blame <file>                      # Show file annotations

# Undoing Changes
git restore <file>                    # Restore file
git restore --staged <file>           # Unstage file
git reset --soft HEAD~1               # Undo last commit (keep changes)
git reset --hard HEAD~1               # Undo last commit (discard changes)
git revert <commit>                   # Create revert commit
```

## Wrapping Up

Git is incredibly powerful, but with great power comes great responsibility. Here are the key takeaways:

### Golden Rules of Git

1. **Commit early and often** - Small, focused commits are better
2. **Write meaningful commit messages** - Your future self will thank you
3. **Use branches for features** - Keep main branch stable
4. **Pull before push** - Stay up to date with remote changes
5. **Don't rewrite shared history** - Use revert instead of reset on shared branches
6. **Test before committing** - Use hooks to enforce quality
7. **Keep repositories clean** - Regular maintenance prevents issues

### Next Steps

- **Practice with personal projects** - The best way to learn Git
- **Explore advanced features** - Submodules, worktrees, and custom scripts
- **Set up automation** - Use hooks and CI/CD for better workflows
- **Learn from others** - Study how successful projects use Git
- **Stay updated** - Git continues to evolve with new features

Remember, mastering Git is a journey, not a destination. Start with the basics, build good habits, and gradually incorporate advanced techniques as you become more comfortable.

The investment in learning Git properly pays enormous dividends in productivity, collaboration, and peace of mind. There's nothing quite like the confidence that comes from knowing your code is safe, versioned, and recoverable!

What Git workflows have worked best for your team? Any Git horror stories or success stories to share? Drop them in the comments below!

---

*May your merges be conflict-free and your commits be meaningful! 🚀*