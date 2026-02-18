# Git -- Scenario-Based Questions

Real-world troubleshooting and problem-solving exercises.

---

### Scenario 1: Accidentally committed to the wrong branch

**Situation:** You made several commits on `main` that should have been on a feature branch.

**How would you fix this?**

**Answer:**

```bash
# 1. Create a new branch at the current position (saves your commits)
git branch feature-x

# 2. Move main back to where it should be
git checkout main
git reset --hard origin/main

# 3. Switch to your feature branch and continue
git checkout feature-x
```

If you have already pushed to `main`, use `git revert` instead of `git reset` to avoid rewriting shared history.

---

### Scenario 2: Need to undo the last commit but keep the changes

**Situation:** You just committed but realize you forgot to add a file or the commit message is wrong.

**How would you fix this?**

**Answer:**

**Option 1 -- Amend the commit (if not pushed):**

```bash
# Fix the message
git commit --amend -m "correct message"

# Add forgotten files
git add forgotten-file.txt
git commit --amend --no-edit
```

**Option 2 -- Soft reset (if not pushed):**

```bash
git reset --soft HEAD~1
# All changes are now staged, ready to recommit
```

**If already pushed:** Use `git revert` or amend + force-push only if no one else has pulled.

---

### Scenario 3: Merge conflict during a pull

**Situation:** `git pull` fails with "CONFLICT (content): Merge conflict in app.py".

**How would you resolve this?**

**Answer:**

```bash
# 1. See which files have conflicts
git status

# 2. Open each conflicted file and resolve manually
#    Remove <<<<<<, ======, >>>>>> markers
#    Keep the correct code

# 3. Stage the resolved files
git add app.py

# 4. Complete the merge
git commit

# Alternative: abort the merge and start over
git merge --abort
```

**Prevention:** Pull with rebase to avoid merge commits:

```bash
git pull --rebase origin main
# If conflict during rebase:
# resolve, then:
git add <file>
git rebase --continue
```

---

### Scenario 4: Accidentally deleted a branch with unmerged work

**Situation:** You ran `git branch -D feature` and lost commits.

**How would you recover it?**

**Answer:**

```bash
# 1. Find the lost commit using reflog
git reflog
# Look for the last commit on the deleted branch

# 2. Recreate the branch at that commit
git branch feature <commit-hash>

# Or checkout directly
git checkout -b feature <commit-hash>
```

This works because `git branch -D` only deletes the pointer, not the commits. Commits stay in the object database until garbage collection (usually 30 days).

---

### Scenario 5: Sensitive data committed to the repository

**Situation:** A developer committed a file containing API keys or passwords to the repo and pushed it.

**How would you fix this?**

**Answer:**

Removing the file in a new commit is **not enough** -- the secret is still in the Git history.

```bash
# 1. Rotate the exposed credentials immediately
#    (this is the most important step)

# 2. Remove the file from the entire history
#    Using git filter-repo (recommended):
pip install git-filter-repo
git filter-repo --path secrets.env --invert-paths

# 3. Force-push the cleaned history
git push --force --all

# 4. All team members must re-clone the repository

# 5. Add the file to .gitignore
echo "secrets.env" >> .gitignore
git add .gitignore
git commit -m "chore: add secrets.env to gitignore"
```

**Prevention:**

- Use `.gitignore` from the start
- Use pre-commit hooks (e.g., `detect-secrets`, `gitleaks`) to block secrets
- Store secrets in environment variables or a vault, never in the repo

---

### Scenario 6: Rebase gone wrong

**Situation:** You rebased a branch and now the history is a mess or tests are failing.

**How would you undo the rebase?**

**Answer:**

```bash
# 1. Find the commit before the rebase started
git reflog
# Look for "rebase (start)" entries
# The commit BEFORE that is your safe point

# 2. Reset to that commit
git reset --hard HEAD@{N}
# where N is the reflog entry number before the rebase
```

If you have already force-pushed the bad rebase:

```bash
git reflog
git reset --hard <pre-rebase-hash>
git push --force
# Notify the team to re-pull
```

---

### Scenario 7: Need to split a large commit into smaller ones

**Situation:** You made one big commit with unrelated changes and need to split it for a cleaner PR.

**How would you do this?**

**Answer:**

```bash
# 1. Interactive rebase to edit the commit
git rebase -i HEAD~1   # or HEAD~N for an older commit
# Change "pick" to "edit" for the target commit

# 2. Reset the commit but keep files
git reset HEAD~1

# 3. Stage and commit in logical groups
git add auth.py
git commit -m "feat: add authentication"

git add api.py
git commit -m "feat: add API routes"

git add tests/
git commit -m "test: add unit tests"

# 4. Continue the rebase
git rebase --continue
```

---

### Scenario 8: Two developers edited the same file, how to avoid conflicts

**Situation:** Your team frequently has merge conflicts in shared files.

**What can you recommend?**

**Answer:**

| Strategy                             | How it helps                                                         |
| ------------------------------------ | -------------------------------------------------------------------- |
| Small, frequent commits              | Less divergence between branches                                     |
| Short-lived branches                 | Merge back to main quickly                                           |
| Rebase before merging                | Catches conflicts early                                              |
| Code ownership / file locking        | Avoid simultaneous edits in critical files                           |
| Modular code design                  | Smaller files with single responsibilities reduce overlap            |
| Communication                        | Coordinate when someone is working on a shared config or schema file |
| Branch protection + required reviews | Forces PRs and catches conflicts before they hit main                |
| Trunk-based development              | Very short-lived branches or direct commits reduce divergence        |

---

### Scenario 9: Need to temporarily switch branches with uncommitted work

**Situation:** You are in the middle of a feature but need to review a PR on another branch urgently.

**How would you handle this?**

**Answer:**

**Option 1 -- Stash:**

```bash
git stash push -m "WIP: login feature"
git checkout pr-branch
# Do the review
git checkout feature
git stash pop
```

**Option 2 -- Worktrees (better for frequent switching):**

```bash
# Create a separate working directory for the PR branch
git worktree add ../pr-review pr-branch
cd ../pr-review
# Do the review in this directory
cd ../main-repo
git worktree remove ../pr-review
```

Worktrees let you have multiple branches checked out simultaneously in different directories.

---

### Scenario 10: CI pipeline shows different results from local

**Situation:** Tests pass locally but fail in CI, or the build works locally but not in CI.

**What are the possible causes?**

**Answer:**

| Cause                             | How to confirm                                | Fix                                                 |
| --------------------------------- | --------------------------------------------- | --------------------------------------------------- |
| Uncommitted or unstaged files     | `git status` shows changes not pushed         | Commit and push everything                          |
| .gitignore excludes a needed file | File exists locally but not in CI             | Remove from .gitignore or use a different approach  |
| Environment differences           | CI uses different OS/version/tools            | Match CI environment locally (use Docker)           |
| Dependency pinning issues         | `package-lock.json` or `go.sum` not committed | Commit lock files                                   |
| Cached build artifacts locally    | Local build reuses old outputs                | Run `git clean -fdx` locally and rebuild            |
| Different branch/commit           | CI runs on a different commit                 | Verify the commit SHA in CI matches what you expect |

---
