---
name: gitlab-issue-creator
description: Use whenever the user wants to create, file, open, log, raise, or post a detailed issue, ticket, task, story, or bug on GitLab — including phrases like "make a gitlab issue for X", "open a ticket about Y", "log this on the taskboard", "create a task in gitlab", or any request that ends with new work being tracked on a GitLab project. The skill interviews the user for context, drafts a full issue body (description, acceptance criteria, tasks, labels, milestone, assignee), gets confirmation, and posts it via `glab` CLI or the GitLab REST API. It also persists the target project and auth choice in a per-repo `.gitlab-taskboard.json` so subsequent invocations skip the setup interview.
---

# GitLab Issue Creator

Create well-structured GitLab issues from a short user description. The skill does two jobs: **one-time setup** for the repo (which GitLab project + how to authenticate), and the **per-issue workflow** (interview → draft → confirm → post).

The goal is high-signal issues — not a one-line "do the thing" — because vague issues create rework. A short interview produces a much better artifact than asking the user to type the whole thing themselves.

## When this triggers

- "Create an issue for…", "open a ticket about…", "log a task on gitlab…"
- "Add this to the taskboard"
- "File a bug for…"
- "Make a story for…"
- Any time the user is describing work that should end up tracked on GitLab

If you're not sure whether the user wants a GitLab issue vs. a local TODO or a GitHub issue, **ask first**. Don't post anything until intent is confirmed.

## Step 1 — Locate or create the repo config

Look for `.gitlab-taskboard.json` in this order:

1. The current working directory
2. A `taskboard/` subdirectory (this user often keeps a separate taskboard repo as a sibling)
3. The repo root (walk up to the nearest `.git` directory)

If found, read it and skip to Step 3.

If not found, run **Step 2** to set it up. Tell the user briefly: "I don't see a taskboard config for this repo yet — let me set that up first." Do this before any issue interview.

## Step 2 — One-time setup interview

Ask the user, using `AskUserQuestion` when available so they can pick rather than type:

1. **GitLab project path or ID**
   - e.g. `mygroup/myrepo` or numeric ID `12345`
   - Try to suggest a default from `git remote get-url origin` if the remote is a GitLab URL — parse out the `group/repo` part and offer it.

2. **GitLab host** (default: `gitlab.com`)
   - Only ask if there's any hint of a self-hosted instance (custom remote host).

3. **Authentication method** — give the user a clear choice:
   - **`glab` CLI** (recommended if installed): uses whatever auth `glab` is already configured with. Check with `glab auth status`. If `glab` isn't installed or not authed, fall back to the token option.
   - **GitLab personal access token**: stored in the `GITLAB_TOKEN` environment variable. **Never write the token into the config file** — only the choice that "we use a token" goes in the file.

Then write the config:

```json
{
  "project": "mygroup/myrepo",
  "host": "gitlab.com",
  "auth": "glab",
  "default_labels": [],
  "default_assignee": null,
  "default_milestone": null
}
```

Confirm with the user before writing, then save to `.gitlab-taskboard.json` at whichever location makes sense (project root by default; if the user has a separate `taskboard/` repo, ask which). Add it to `.gitignore` only if it ever contains anything sensitive — by design it doesn't, so usually commit it.

### Verifying auth works

- **glab path**: run `glab auth status` (no token printed). If it fails, tell the user how to fix it (`glab auth login`) and stop.
- **token path**: check `$env:GITLAB_TOKEN` (PowerShell) or `$GITLAB_TOKEN` (bash). If empty, instruct the user to set it for the session and re-invoke. Do not prompt for the token directly and never echo it.

## Step 3 — Interview the user for the issue

Now gather the details. Don't dump a 10-question survey — work in 2–3 small rounds, using `AskUserQuestion` for choices and free text only when needed:

**Round 1 — the essentials**
- One-sentence summary (becomes the title)
- Type: bug / feature / task / chore / spike (controls default labels)
- Short context: what's the problem or goal? what's the user-visible impact?

**Round 2 — the body**
- Acceptance criteria: what does "done" look like? Ask for 2–5 bullets. If the user gives one fuzzy line, propose 2–3 concrete bullets and ask them to confirm or edit.
- Tasks/subtasks: optional checklist of the steps to do it. Skip if the work is small.

**Round 3 — metadata**
- Labels (suggest defaults from the config + type)
- Milestone (offer current open milestones from `glab issue milestone list` if available)
- Assignee (default to the user themselves if their GitLab username is known, otherwise leave unassigned)

Be willing to skip rounds when the user has already given the info up-front in their original prompt. Don't re-ask.

## Step 4 — Draft, confirm, post

Render the draft using this template. Use it exactly — consistent structure makes issues skimmable:

```markdown
## Description
<2–6 sentences of context. What is the problem / goal? Who is affected? Why now?>

## Acceptance Criteria
- [ ] <criterion 1>
- [ ] <criterion 2>
- [ ] <criterion 3>

## Tasks
- [ ] <step 1>
- [ ] <step 2>

## Notes / References
<links, related issues, screenshots, prior discussion — omit the heading if empty>
```

Show the rendered draft to the user (title + body + labels + milestone + assignee) and ask: "Post this as-is, or edit first?" Only post after explicit confirmation. Posting an issue is visible to teammates — it's not a reversible local edit.

### Posting via `glab`

```bash
glab issue create \
  --repo <project> \
  --title "<title>" \
  --description "$(cat <<'EOF'
<body>
EOF
)" \
  --label "<comma,separated,labels>" \
  --milestone "<milestone>" \
  --assignee "<username>"
```

Omit flags whose value is empty. On PowerShell, use a here-string (`@'…'@`) for the body — see "Body passing" below.

### Posting via REST API (token mode)

```bash
curl --request POST \
  --header "PRIVATE-TOKEN: $GITLAB_TOKEN" \
  --header "Content-Type: application/json" \
  --data @issue.json \
  "https://<host>/api/v4/projects/<URL-encoded-project>/issues"
```

Where `issue.json` is `{"title": "...", "description": "...", "labels": "a,b", "milestone_id": <id>, "assignee_ids": [<id>]}`. URL-encode the project path: `mygroup/myrepo` → `mygroup%2Fmyrepo`. Resolve milestone titles to IDs first via `GET /projects/:id/milestones?state=active` if needed.

### Body passing on PowerShell

PowerShell doesn't accept bash here-docs. Use a single-quoted here-string and pass via `--description-file`:

```powershell
@'
## Description
…
'@ | Set-Content -Encoding utf8 issue-body.md
glab issue create --repo <project> --title "<title>" --description-file issue-body.md
Remove-Item issue-body.md
```

Single-quoted (`@'…'@`) preserves `$` and backticks literally — important if the body contains code snippets.

## Step 5 — Report back

After posting, show the user:
- The issue URL (the `glab` / API response includes it)
- The issue ID (e.g. `#42`)

Offer follow-ups only when relevant: "Want me to add this to a progress doc?" — don't volunteer unrelated work.

## Failure modes & recovery

| Symptom | Cause | Fix |
|---|---|---|
| `glab: command not found` | not installed | Offer the token path, or tell user to install glab |
| `glab auth status` shows not logged in | glab not configured | Run `glab auth login`; if user picks token, fall back |
| API returns 401 | bad / missing token | Re-check `GITLAB_TOKEN`; never print the token |
| API returns 404 on project | wrong project path or no access | Re-confirm project; check user has Developer+ role |
| Milestone name doesn't match | typo or closed milestone | List active milestones, ask user to pick |
| User cancels at confirm | by design | Don't post; save the draft to a temp file so they can resume |

## Worked example

User says: *"create a gitlab issue — the FD product list page is loading slowly when there are 500+ products, we need to paginate it"*

The finished draft should look roughly like this:

```markdown
Title: Paginate FD product list to fix slow load with 500+ products

## Description
The FD product list page (`/admin/fd-products`) currently loads all products in
a single query. With 500+ active products in production, initial render takes
8–12s and admins are reporting timeouts. We need server-side pagination plus a
visible page selector so the page stays responsive at any list size.

## Acceptance Criteria
- [ ] List page loads in under 1s with 1,000+ products in the table
- [ ] Page size defaults to 25 with a 25/50/100 selector
- [ ] Page state survives back/forward navigation (in URL query string)
- [ ] No regression in existing search/filter behaviour

## Tasks
- [ ] Add `page` + `per_page` params to the FD products list endpoint
- [ ] Wire pagination component into the admin Vue page
- [ ] Update integration test for the endpoint
- [ ] Smoke-test with a seeded 1k-row dataset

Labels: ~bug ~backend ~frontend
Milestone: Sprint 24
Assignee: @dhiraj
```

Use this as the bar for quality — concrete numbers, concrete file paths, AC that someone else could verify, tasks small enough to commit against.

## Principles

- **Never post without explicit confirmation.** Issues are visible to teammates.
- **Never store tokens in the config file.** Only the choice that we use a token.
- **Interview is short, not exhaustive.** 2–3 rounds, skip rounds the user already answered.
- **Default to the template.** Consistency matters more than custom structure per-issue.
- **One issue per invocation** unless the user explicitly says "create three issues".
- **Don't volunteer side-quests.** Post the issue, report back, stop.
