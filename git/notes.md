# Git -- Notes & Cheat Sheet

Quick-reference commands and concepts for revision.

---

## Configuration

```bash
# Set identity
git config --global user.name "Your Name"
git config --global user.email "you@example.com"

# Set default branch name
git config --global init.defaultBranch main

# Set default editor
git config --global core.editor vim

# Set rebase as default pull strategy
git config --global pull.rebase true

# View all config
git config --list --show-origin
```

---

## Daily Workflow

```bash
# Clone
git clone <url>
git clone <url> <directory>
git clone --depth 1 <url>           # shallow clone (latest commit only)

# Status and diff
git status
git status -s                       # short format
git diff                            # unstaged changes
git diff --staged                   # staged changes
git diff main..feature              # diff between branches

# Stage
git add <file>
git add .                           # stage all
git add -p                          # interactive hunk staging

# Commit
git commit -m "message"
git commit -am "message"            # stage tracked files + commit
git commit --amend --no-edit        # add to last commit

# Push
git push
git push -u origin feature          # set upstream
git push --force-with-lease         # safer force push
```

---

## Branching

```bash
# Create and switch
git switch -c feature               # Git 2.23+
git checkout -b feature             # classic

# List
git branch                          # local
git branch -r                       # remote
git branch -a                       # all
git branch --merged                 # merged into current
git branch --no-merged              # not yet merged

# Delete
git branch -d feature               # safe (must be merged)
git branch -D feature               # force

# Rename
git branch -m old-name new-name

# Delete remote branch
git push origin --delete feature
```

---

## Merging and Rebasing

```bash
# Merge
git merge feature
git merge --no-ff feature           # always create merge commit
git merge --squash feature          # squash all commits into one
git merge --abort                   # cancel a conflicted merge

# Rebase
git rebase main                     # replay current branch onto main
git rebase -i HEAD~3                # interactive rebase (last 3 commits)
git rebase --abort                  # cancel
git rebase --continue               # after resolving conflicts
```

---

## Undoing Changes

```bash
# Discard working directory changes
git checkout -- <file>              # classic
git restore <file>                  # Git 2.23+

# Unstage a file
git reset HEAD <file>               # classic
git restore --staged <file>         # Git 2.23+

# Reset (move branch pointer)
git reset --soft HEAD~1             # undo commit, keep staged
git reset HEAD~1                    # undo commit, keep unstaged
git reset --hard HEAD~1             # undo commit, discard everything

# Revert (safe for shared branches)
git revert <hash>
git revert -m 1 <merge-hash>       # revert a merge commit

# Recover lost commits
git reflog
git reset --hard HEAD@{N}
```

---

## Remote Operations

```bash
# Remotes
git remote -v
git remote add upstream <url>
git remote remove <name>
git remote set-url origin <new-url>

# Fetch and integrate
git fetch origin
git fetch --all
git fetch --prune                   # clean up deleted remote branches
git pull --rebase origin main

# Push
git push origin main
git push --all                      # push all branches
git push --tags                     # push all tags
git push --force-with-lease         # safe force push
```

---

## Inspection

```bash
# Log
git log --oneline -10               # last 10 commits, compact
git log --graph --oneline --all     # visual branch graph
git log --author="name"             # filter by author
git log --since="2 weeks ago"       # filter by date
git log -- <file>                   # history of a specific file
git log -p -- <file>                # show diffs for a file

# Blame
git blame <file>                    # who changed each line
git blame -L 10,20 <file>           # lines 10-20 only

# Show
git show <hash>                     # show a specific commit
git show <hash>:<file>              # show a file at a specific commit

# Search
git log -S "function_name"          # find commits that add/remove a string
git log --grep="bug fix"            # search commit messages
git grep "pattern"                  # search working directory
```

---

## Tags

```bash
git tag v1.0.0                      # lightweight
git tag -a v1.0.0 -m "Release 1.0" # annotated
git tag -a v1.0.0 <hash>            # tag a past commit
git tag -l "v1.*"                   # list matching tags
git push origin v1.0.0              # push single tag
git push origin --tags              # push all tags
git tag -d v1.0.0                   # delete local
git push origin --delete v1.0.0     # delete remote
```

---

## Stash

```bash
git stash                           # stash changes
git stash push -m "description"     # with message
git stash -u                        # include untracked
git stash list                      # list all stashes
git stash pop                       # apply and remove
git stash apply                     # apply and keep
git stash apply stash@{2}           # apply specific
git stash drop stash@{0}            # remove specific
git stash clear                     # remove all
```

---

## Useful Aliases

```bash
git config --global alias.st "status -s"
git config --global alias.co "checkout"
git config --global alias.br "branch"
git config --global alias.ci "commit"
git config --global alias.lg "log --oneline --graph --all"
git config --global alias.last "log -1 HEAD"
git config --global alias.unstage "restore --staged"
```

---

## Key Concepts Summary

| Concept | One-line explanation |
|---------|---------------------|
| HEAD | Pointer to the current branch or commit |
| Detached HEAD | HEAD points to a commit, not a branch |
| Fast-forward merge | Target branch has no new commits; pointer simply moves forward |
| Three-way merge | Both branches have new commits; creates a merge commit |
| Upstream | The default remote branch your local branch tracks |
| Origin | Default name for the remote you cloned from |
| Shallow clone | Clone with limited history (`--depth`) |
| Worktree | Multiple working directories sharing one .git |
| Reflog | Local log of all HEAD movements |
| Object types | blob (file), tree (directory), commit, tag |

---
