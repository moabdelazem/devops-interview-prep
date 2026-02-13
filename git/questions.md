# Git -- Interview Questions

General and conceptual interview questions with clear answers.

---

## Fundamentals

### Q1: What is Git and how does it differ from other version control systems?

**Answer:**

Git is a distributed version control system (DVCS). Every developer has a full copy of the repository history on their local machine, including all branches and commits.

**How it differs from centralized VCS (SVN, CVS):**

| Feature | Git (distributed) | SVN (centralized) |
|---------|-------------------|-------------------|
| Repository copy | Full history on every machine | Only the server has full history |
| Offline work | Full commit, branch, log, diff offline | Requires server connection |
| Branching | Lightweight, instant (pointer to a commit) | Heavyweight (copies directory tree) |
| Speed | Fast (everything is local) | Slower (network round-trips) |
| Single point of failure | No (every clone is a backup) | Yes (server goes down, work stops) |

---

### Q2: Explain the three areas in Git (working directory, staging area, repository).

**Answer:**

Git tracks files through three areas:

| Area | What it is | Key command |
|------|-----------|-------------|
| **Working directory** | Your actual files on disk. Modifications here are "unstaged". | `git status` shows changes |
| **Staging area (index)** | A snapshot of what will go into the next commit. Files added here are "staged". | `git add <file>` moves files here |
| **Repository (.git)** | Permanent history of committed snapshots. | `git commit` saves staged changes here |

**Flow:**

```
Edit file --> Working Directory (modified)
    |
git add --> Staging Area (staged)
    |
git commit --> Repository (committed)
```

You can selectively stage parts of a file using `git add -p`, which lets you choose individual hunks to include in the next commit.

---

### Q3: What is a commit and what does it contain?

**Answer:**

A commit is a snapshot of the entire repository at a point in time. It contains:

- **Tree object** -- A snapshot of all tracked files and directories
- **Parent pointer** -- Reference to the previous commit (merge commits have two parents)
- **Author** -- Who wrote the change (name, email, timestamp)
- **Committer** -- Who applied the commit (can differ in rebase/cherry-pick)
- **Message** -- Description of the change
- **SHA-1 hash** -- A 40-character unique identifier computed from all of the above

Commits are immutable. Commands like `git commit --amend` or `git rebase` do not modify commits -- they create new commits with new hashes.

---

### Q4: What is the difference between `git merge` and `git rebase`?

**Answer:**

Both integrate changes from one branch into another, but they do it differently.

**Merge:** Creates a new merge commit that ties two branches together. Preserves the full branch history.

```bash
git checkout main
git merge feature
```

```
Before:     main: A - B - C
            feature:   D - E

After:      main: A - B - C - M  (M is the merge commit)
                       \       /
                        D - E
```

**Rebase:** Replays your branch's commits on top of the target branch. Creates a linear history but rewrites commit hashes.

```bash
git checkout feature
git rebase main
```

```
Before:     main: A - B - C
            feature:   D - E

After:      main: A - B - C
            feature:       D' - E'  (new hashes, same changes)
```

**When to use each:**

| Situation | Use |
|-----------|-----|
| Shared/public branches (main, develop) | Merge -- never rewrite shared history |
| Local feature branches before a PR | Rebase -- keeps history clean |
| Resolving divergent history | Merge -- safer, preserves context |

**Golden rule:** Never rebase commits that have been pushed to a shared branch.

---

### Q5: What is a branch in Git?

**Answer:**

A branch is a lightweight, movable pointer to a commit. It is not a copy of files -- it is just a 41-byte file containing a commit hash.

```bash
# Create a branch
git branch feature

# Create and switch
git checkout -b feature
# or (Git 2.23+)
git switch -c feature

# List branches
git branch -a          # local and remote
git branch --merged    # branches merged into current

# Delete a branch
git branch -d feature  # safe delete (only if merged)
git branch -D feature  # force delete
```

**HEAD** is a special pointer to the branch you are currently on. When you commit, HEAD (and the current branch) move forward to the new commit.

**Detached HEAD:** If you checkout a commit hash directly (`git checkout abc123`), HEAD points to a commit instead of a branch. Commits made here are lost unless you create a branch.

---

### Q6: What is the difference between `git fetch`, `git pull`, and `git push`?

**Answer:**

| Command | What it does | Changes local files? |
|---------|-------------|---------------------|
| `git fetch` | Downloads new commits and refs from the remote. Updates remote-tracking branches (`origin/main`). | No |
| `git pull` | `git fetch` + `git merge` (or `git rebase` with `--rebase`). Downloads and integrates changes. | Yes |
| `git push` | Uploads local commits to the remote. | No (changes the remote) |

**Best practice:** Use `git fetch` + `git merge` (or `git rebase`) separately instead of `git pull` to see what changed before integrating.

```bash
git fetch origin
git log HEAD..origin/main   # see what is new on the remote
git merge origin/main       # integrate when ready
```

---

### Q7: What is a Git conflict and how do you resolve it?

**Answer:**

A conflict happens when two branches modify the same lines in a file and Git cannot automatically merge them.

**What it looks like:**

```
<<<<<<< HEAD
current branch's version of the code
=======
incoming branch's version of the code
>>>>>>> feature-branch
```

**How to resolve:**

```bash
# 1. Open the conflicted file(s) and edit manually
#    Remove the markers and keep the correct code

# 2. Mark as resolved
git add <file>

# 3. Complete the merge
git commit
```

**Tools:**

```bash
git mergetool         # opens configured merge tool (vimdiff, meld, etc.)
git diff --check      # shows remaining conflict markers
```

**How to prevent conflicts:**
- Pull and rebase frequently
- Keep branches short-lived
- Communicate with the team about shared files

---

### Q8: What does `git stash` do?

**Answer:**

`git stash` saves your uncommitted changes (staged and unstaged) to a stack and reverts the working directory to a clean state. Useful when you need to switch branches without committing half-done work.

```bash
# Stash current changes
git stash

# Stash with a message
git stash push -m "work in progress on login feature"

# Include untracked files
git stash -u

# List all stashes
git stash list

# Apply the most recent stash (keeps it in the stack)
git stash apply

# Apply and remove from stack
git stash pop

# Apply a specific stash
git stash apply stash@{2}

# Drop a specific stash
git stash drop stash@{0}

# Clear all stashes
git stash clear
```

---

### Q9: What is `git cherry-pick`?

**Answer:**

`git cherry-pick` applies a specific commit from one branch onto another. It creates a new commit with the same changes but a different hash.

```bash
git cherry-pick <commit-hash>

# Cherry-pick without committing (stage changes only)
git cherry-pick --no-commit <hash>

# Cherry-pick a range
git cherry-pick A..B     # excludes A, includes B
git cherry-pick A^..B    # includes A and B
```

**Use cases:**
- Backporting a bug fix from `main` to a release branch
- Pulling a single commit from a feature branch that is not ready to merge

---

### Q10: What is `git reset` and what are the three modes?

**Answer:**

`git reset` moves the HEAD (and current branch pointer) to a specified commit. The three modes control what happens to the staging area and working directory.

| Mode | HEAD | Staging area | Working directory | Use case |
|------|------|-------------|-------------------|----------|
| `--soft` | Moves | Unchanged | Unchanged | Undo a commit but keep all changes staged |
| `--mixed` (default) | Moves | Reset to match commit | Unchanged | Undo a commit and unstage changes |
| `--hard` | Moves | Reset | Reset | Completely discard commits and all changes |

```bash
# Undo last commit, keep changes staged
git reset --soft HEAD~1

# Undo last commit, unstage changes
git reset HEAD~1

# Discard everything (DESTRUCTIVE)
git reset --hard HEAD~1

# Unstage a file (without moving HEAD)
git reset HEAD <file>
```

**`git reset` vs `git revert`:**

| Command | What it does | Safe for shared branches? |
|---------|-------------|--------------------------|
| `git reset` | Removes commits from history | No -- rewrites history |
| `git revert` | Creates a new commit that undoes a previous commit | Yes -- adds to history |

---

### Q11: What is `.gitignore` and how does it work?

**Answer:**

`.gitignore` tells Git which files and directories to exclude from tracking. It uses glob patterns.

**Common patterns:**

```gitignore
# Compiled files
*.o
*.pyc
__pycache__/

# Dependencies
node_modules/
vendor/

# Environment files
.env
.env.local

# IDE files
.vscode/
.idea/

# OS files
.DS_Store
Thumbs.db

# Build output
dist/
build/
*.jar

# Logs
*.log
```

**Rules:**
- Patterns are relative to the `.gitignore` file location
- A trailing `/` matches only directories
- `!` negates a pattern (force-include a file)
- `#` is a comment
- `**` matches any number of directories (`**/logs` matches `logs`, `a/logs`, `a/b/logs`)

**Already tracked files:** `.gitignore` only prevents untracked files from being added. To stop tracking a file that is already committed:

```bash
git rm --cached <file>     # remove from Git, keep on disk
echo "<file>" >> .gitignore
git commit -m "stop tracking <file>"
```

---

### Q12: What are Git tags and when do you use them?

**Answer:**

Tags mark specific commits as important, typically for releases.

**Lightweight tags:** Just a pointer to a commit (like a branch that does not move).

```bash
git tag v1.0.0
```

**Annotated tags:** A full Git object with tagger name, date, message, and optional GPG signature. Preferred for releases.

```bash
git tag -a v1.0.0 -m "Release version 1.0.0"
```

**Common commands:**

```bash
# List tags
git tag -l
git tag -l "v1.*"        # filter by pattern

# Tag a past commit
git tag -a v1.0.0 <commit-hash> -m "Release 1.0.0"

# Push a single tag
git push origin v1.0.0

# Push all tags
git push origin --tags

# Delete a local tag
git tag -d v1.0.0

# Delete a remote tag
git push origin --delete v1.0.0
```

---

## Collaboration and Workflows

### Q13: What is the difference between `git clone`, `git fork`, and `git remote`?

**Answer:**

| Concept | What it does |
|---------|-------------|
| `git clone` | Copies a remote repository to your local machine. Sets up `origin` as the remote. |
| Fork | A server-side copy of a repository under your account (GitHub/GitLab feature, not a Git command). |
| `git remote` | Manages named references to remote repositories. |

**Typical fork workflow:**

```bash
# 1. Fork on GitHub (creates your-user/repo)
# 2. Clone your fork
git clone git@github.com:your-user/repo.git

# 3. Add the original repo as upstream
git remote add upstream git@github.com:original-user/repo.git

# 4. Keep in sync
git fetch upstream
git merge upstream/main
```

```bash
# Manage remotes
git remote -v                          # list remotes
git remote add <name> <url>            # add
git remote remove <name>               # remove
git remote set-url origin <new-url>    # change URL
```

---

### Q14: Describe the Git Flow branching strategy.

**Answer:**

Git Flow defines a strict branch model for release management:

| Branch | Purpose | Lifetime |
|--------|---------|----------|
| `main` | Production-ready code. Every commit is a release. | Permanent |
| `develop` | Integration branch for features. Next release candidate. | Permanent |
| `feature/*` | New features. Branched from `develop`, merged back to `develop`. | Temporary |
| `release/*` | Preparing a release. Bug fixes only, no new features. Merged to `main` and `develop`. | Temporary |
| `hotfix/*` | Urgent production fixes. Branched from `main`, merged to `main` and `develop`. | Temporary |

```
main:      v1.0 --------- v1.0.1 ---------- v1.1
              \             /    \              /
hotfix:        \       hotfix-1   \            /
                \                  \          /
develop:     ----+--+--+--+--------+--+--+--+--
                  \   /                \   /
feature:       feat-A               feat-B
```

**Alternatives commonly asked about:**

| Strategy | Key idea |
|----------|---------|
| **GitHub Flow** | Single `main` branch + feature branches + pull requests. Simpler. |
| **Trunk-Based** | Everyone commits to `main` (or via short-lived branches). CI/CD focused. |
| **GitLab Flow** | Like GitHub Flow but with environment branches (`staging`, `production`). |

---

### Q15: What is a pull request (merge request)?

**Answer:**

A pull request (GitHub) or merge request (GitLab) is a request to merge changes from one branch into another. It is a collaboration and code review mechanism, not a Git command.

**What a PR includes:**
- The diff of changes
- A description of what was changed and why
- Code review comments and approvals
- CI/CD pipeline results (automated tests, linting)

**Best practices:**
- Keep PRs small and focused on a single concern
- Write a clear description with context for reviewers
- Reference the issue or ticket number
- Ensure CI passes before requesting review
- Squash commits before merging if the history is messy

---

### Q16: How do you undo a pushed commit safely?

**Answer:**

Use `git revert` -- it creates a new commit that inverts the changes of a previous commit without rewriting history.

```bash
# Revert a single commit
git revert <commit-hash>

# Revert without committing (useful for reverting multiple)
git revert --no-commit <hash1>
git revert --no-commit <hash2>
git commit -m "revert: undo feature X"

# Revert a merge commit (must specify which parent with -m)
git revert -m 1 <merge-commit-hash>
```

**Never use `git reset --hard` + `git push --force` on shared branches** unless you have coordinated with everyone. Force-pushing rewrites history and causes problems for anyone who has already pulled.

---

### Q17: What is `git bisect`?

**Answer:**

`git bisect` performs a binary search through commit history to find the commit that introduced a bug.

```bash
# Start bisecting
git bisect start

# Mark the current commit as bad (has the bug)
git bisect bad

# Mark a known good commit (did not have the bug)
git bisect good <commit-hash>

# Git checks out a commit in the middle. Test it.
# If the bug is present:
git bisect bad
# If the bug is absent:
git bisect good

# Repeat until Git identifies the first bad commit.

# When done, return to original state
git bisect reset
```

You can also automate it with a test script:

```bash
git bisect start HEAD v1.0.0
git bisect run ./test-script.sh
```

Git will automatically run the script on each step and mark commits as good (exit 0) or bad (exit 1).

---

### Q18: What is `git reflog` and when is it useful?

**Answer:**

`git reflog` records every time HEAD moves -- commits, resets, rebases, checkouts, amends. It is a local safety net that lets you recover "lost" commits.

```bash
# View reflog
git reflog

# Output example:
# abc1234 HEAD@{0}: commit: add login feature
# def5678 HEAD@{1}: reset: moving to HEAD~2
# 789abcd HEAD@{2}: commit: old commit that was reset away

# Recover a commit after a hard reset
git reset --hard HEAD@{2}

# Or create a branch from the lost commit
git branch recovery HEAD@{2}
```

**Key facts:**
- Reflog is local only (not shared via push/pull)
- Entries expire after 90 days (configurable via `gc.reflogExpire`)
- It tracks HEAD movements, not file changes

---
