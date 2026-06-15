---
name: files-to-deploy
description: Use whenever the user wants to know which files changed, were added, or were modified between two git commits, or from a commit up to HEAD — for deployment, code review, or tracking purposes. Triggers on phrases like "files to deploy", "what changed between commits", "files changed since commit", "deploy list", "changed files between X and Y", "which files were modified since commit", "get changed files for deploy", "what needs to be deployed". Always use this skill before the user has to manually run git diff commands.
---

# Files to Deploy

Identify all files that have been added, modified, copied, or renamed between two points in a git history — useful for selective deployments, manual FTP uploads, or understanding the scope of a release.

## Step 1 — Ask the user which mode they want

Use `AskUserQuestion` if available, otherwise ask directly:

> "Do you want files changed between two specific commits, or from a specific commit up to the latest (HEAD)?"

**Option A — Between two commits:** User provides both a start commit and an end commit.  
**Option B — Commit to HEAD:** User provides only the starting commit; the endpoint is the current HEAD.

## Step 2 — Collect the commit references

Based on their choice:

**For Option A (two commits):**
- Ask: "What is the starting commit hash (or tag/branch name)?"
- Ask: "What is the ending commit hash (or tag/branch name)?"

**For Option B (commit to HEAD):**
- Ask: "What is the starting commit hash (or tag/branch name)?"
- The end point is `HEAD`.

**Tips for the user if they're unsure of hashes:**
- Run `git log --oneline -20` to list recent commits with their short hashes
- They can also use branch names (e.g. `main`, `release/v2.1`) or tags (e.g. `v1.0.0`)
- Offer to run `git log --oneline -20` for them if they ask

## Step 3 — Run the git diff command

Use `--name-only` to get a plain list of file paths — no status letters, no decoration. Deleted files are excluded via `--diff-filter=ACMR` since they don't need to be uploaded.

**Option A (two commits):**
```bash
git diff --name-only --diff-filter=ACMR <start_commit> <end_commit>
```

**Option B (commit to HEAD):**
```bash
git diff --name-only --diff-filter=ACMR <start_commit> HEAD
```

For renamed files, only the new (destination) path is included — that's the file that needs to be deployed.

## Step 4 — Present the results

Before the file list, run and display:

```bash
git remote -v
git branch --show-current
```

Show these as context so the user knows exactly which remote and branch they're deploying from:

```
Remote:  origin  https://github.com/org/repo.git (fetch)
         origin  https://github.com/org/repo.git (push)
Branch:  main
```

Then output the file list exactly as returned by git: one path per line, no headers, no labels, no counts. This is the deploy list — keep it clean so it can be copied directly into a deploy script or tool.

```
application/modules/backend/product/controllers/Product.php
application/modules/backend/product/views/_list.php
assets/frontend/css/main.css
```

## Step 5 — Offer to save the list

After showing the results, ask: "Would you like me to save this to `files-to-deploy.txt`?"

If yes, write the same plain list to `files-to-deploy.txt` in the current working directory.

## Edge cases

- **No changes found:** Tell the user "No files were added or modified between those commits." Then suggest verifying the commit hashes with `git log --oneline`.
- **Invalid commit hash:** If the git command errors, show the error and ask the user to double-check the commit reference. Offer to run `git log --oneline -20` to help them find the right hash.
- **Large output (50+ files):** Output the full plain list as-is — don't summarize or group. The user needs the complete list for deployment.
