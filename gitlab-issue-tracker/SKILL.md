---
name: gitlab-issue-tracker
description: Use whenever the user references a GitLab issue by its short ID (e.g. "#5", "#12", "issue 7", "task #3") and wants work done against it — phrases like "work on #5", "start #12", "execute issue 7", "implement #3 and #4", "pick up #9", "let's tackle the next issue", or any agent/subagent dispatch that should be tied to a GitLab issue's lifecycle. The skill takes ownership of the issue's state: it parses the ID, fetches the issue via `glab` CLI or the GitLab REST API (using the project's `.gitlab-taskboard.json` config), moves it to "In Progress", auto-ticks acceptance-criteria checkboxes as the work that satisfies each one is completed (announcing each tick), and after all AC are met asks the user to confirm closing — then closes the issue and posts a short summary comment. Use it as a wrapper around any execution flow so the GitLab board stays a faithful mirror of what the agent actually did.
---

# GitLab Issue Tracker

This skill ties the GitLab board to whatever execution flow the model is running — direct edits, an `Agent` subagent, a long-running implementation, whatever. The board is supposed to reflect reality. Without active tracking, issues stay open after the work is done or sit in "Open" while engineers are halfway through. This skill closes that gap.

Companion to `gitlab-issue-creator`: that skill creates issues, this one runs them. They share the same `.gitlab-taskboard.json` config and auth choice.

## When this triggers

The user references a GitLab issue ID in short form (`#5`, `#12`) and the conversation involves doing work, not just reading or discussing the issue.

Examples that should trigger:
- "start working on #5"
- "implement #12 and #13"
- "execute task #7"
- "pick up #9 next"
- "let's do #3 — dispatch a subagent"

Examples that should *not* trigger (read-only):
- "show me what's in #5"
- "what does issue 12 say?"
- "summarise the acceptance criteria on #7"

If unsure, ask: "Do you want me to actually do this work and track the issue, or just read it?"

## Step 0 — Confirm config exists

Look for `.gitlab-taskboard.json` (same precedence as the creator skill: cwd → `taskboard/` → repo root). If missing, tell the user: "I don't see a taskboard config for this repo — please run `/skill-creator gitlab-issue-creator` first, or set one up." Don't try to recreate the setup logic here; it lives in the creator skill.

Read the config to know:
- `project` — group/repo path or numeric ID
- `host` — defaults to `gitlab.com`
- `auth` — `glab` or `token`

Verify auth is usable (`glab auth status` for the glab path; `$env:GITLAB_TOKEN` non-empty for the token path). Fail fast if not.

## Step 1 — Parse the issue ID(s)

The user types IDs in human form: `#5`, `#12`, `issue 7`, `the third one (#9)`. Strip the `#` and any prose. Accept multiple IDs in one prompt.

If the user says something vague like "pick up the next issue", offer to list open issues in the project (`glab issue list --repo <project> --label "Open" --state opened`) and have them pick. Don't guess.

## Step 2 — Fetch and announce

For each issue ID, fetch the full issue:

```bash
glab issue view <id> --repo <project> --output json
```

Or via REST:

```bash
curl --header "PRIVATE-TOKEN: $GITLAB_TOKEN" \
  "https://<host>/api/v4/projects/<URL-encoded-project>/issues/<iid>"
```

Show the user a one-paragraph summary: title, current labels, AC count (e.g. "5 acceptance criteria, 2 already ticked"), and milestone if any. This anchors the work — the user (and you) should know what "done" means before touching anything.

If the issue is already closed: stop and ask "this issue is closed already — reopen and continue, or pick a different one?"

## Step 3 — Move to "In Progress"

Add the `In Progress` label (this project's convention — see `gitlab-progress-sync` for the same pattern):

```bash
glab issue update <id> --repo <project> --label "In Progress"
```

REST: `PUT /projects/:id/issues/:iid` with `add_labels=In Progress`.

Announce briefly: "#5 → In Progress". Don't be verbose.

## Step 4 — Execute the work

Now the model does the work. This is mostly the model's normal flow (Edit, Write, Agent, whatever). The skill's job during execution is to **map progress to AC items** as work completes.

### How AC ticking works

Acceptance criteria live in the issue body as markdown checkboxes:

```markdown
## Acceptance Criteria
- [ ] List page loads in under 1s with 1,000+ products
- [ ] Page size defaults to 25 with a 25/50/100 selector
- [ ] Page state survives back/forward navigation
- [ ] No regression in existing search/filter behaviour
```

Each item is independent and verifiable. As you finish work that fulfils one of them, tick that line. The mechanism:

1. Fetch the current issue body fresh (don't reuse a stale copy — a teammate may have edited it).
2. Find the exact line `- [ ] <criterion>` and replace with `- [x] <criterion>`.
3. Push the updated body back via `glab issue update --description-file` or REST `PUT` with `description`.
4. Announce: "Ticked AC #2 (page size selector) — added the 25/50/100 dropdown in `FdProductList.vue:42`."

The announcement matters. It lets the user catch mis-ticks early. If the model ticks the wrong item, the user can say "untick AC #2" and the skill should un-tick it.

### When to tick — and when not to

Tick when: a unit of work that the AC describes is functionally complete and verifiable. If AC says "tests pass", run the tests first.

Don't tick when:
- The work is partial ("I added the dropdown but haven't wired the handler yet").
- The work is speculative ("this should make the page faster" — measure first if the AC has a number).
- The AC mentions something outside this session's scope ("deployed to staging" when we haven't deployed).

When in doubt, leave it unticked and tell the user why: "AC #4 mentions 'no regression in search' — I haven't run the search test suite. Want me to run it before ticking?"

### Multiple issues in flight

If the user said "work on #12 and #13", treat each issue's AC list independently. Don't cross-tick. Announcements should always include the issue ID: "Ticked #13 AC #1".

### Issue body has no AC checkboxes

Some issues are written without a formal AC list. Don't fabricate one. Skip ticking entirely and just rely on Step 5's close confirmation — the user can confirm "done" verbally.

## Step 5 — Close with confirmation

When all AC items on an issue are ticked (or for issues without AC, when the user says the work is done), ask:

> "All 4 AC met on #5 — close it now?"

On yes, do two things:

1. **Post a short summary comment** (1–3 sentences) — what was changed, where, and any commit/PR links the model knows. This is the audit trail teammates use to understand what shipped without re-reading the diff.

   ```bash
   glab issue note create <id> --repo <project> --message "Closed: implemented pagination in FdProductListController.php and FdProductList.vue. Page size selector at 25/50/100, URL-synced state. See commit abc1234."
   ```

   REST: `POST /projects/:id/issues/:iid/notes` with `body`.

2. **Close the issue**:

   ```bash
   glab issue update <id> --repo <project> --state-event close --unlabel "In Progress"
   ```

   REST: `PUT /projects/:id/issues/:iid` with `state_event=close` and `remove_labels=In Progress`.

Announce: "#5 closed, comment posted."

If the user says no / not yet: leave the issue in "In Progress" with the current AC state. Don't pester.

## Step 6 — Partial / interrupted sessions

If the user wraps up mid-flight ("park this, I'll come back tomorrow"):

- Leave the issue in "In Progress".
- Leave AC ticks as they stand — they reflect real progress.
- Optionally post a brief progress note: "Paused — 2/4 AC done. Remaining: <list>." Only do this if the user asks or if the session looks like it'll be a long gap.
- Do **not** auto-close.

## Worked example

User: *"start #5 — dispatch a subagent if it helps"*

1. Read `.gitlab-taskboard.json` → project = `mygroup/myrepo`, auth = `glab`.
2. `glab issue view 5 --repo mygroup/myrepo --output json` → title "Paginate FD product list", 4 AC, all unticked, labels: `bug`, `backend`, `frontend`.
3. Summarise: "#5 'Paginate FD product list' — 4 AC, all open. Moving to In Progress."
4. `glab issue update 5 --repo mygroup/myrepo --label "In Progress"`.
5. Spawn `Agent` (or work directly) on the implementation.
6. After backend changes land + endpoint test passes: tick AC #1, announce.
7. After frontend dropdown wired + state in URL: tick AC #2 and #3 together (one fetch, one update), announce.
8. Run search test suite, passes: tick AC #4, announce.
9. Ask: "All 4 AC met — close #5?" → yes.
10. Post summary note, close issue, unlabel In Progress.

## Failure modes

| Symptom | Cause | Fix |
|---|---|---|
| `glab` not installed / not logged in | env issue | Fall back to token path if config allows, else stop and tell user |
| Issue ID 404 | wrong project or wrong ID | Confirm project in config matches; ask user to recheck the ID |
| Description update returns 409 / conflict | teammate edited in parallel | Re-fetch body, re-apply tick, re-PUT |
| AC line not found when ticking | issue body changed shape | Re-fetch, show user the current AC list, ask which one they meant |
| Token 401 | bad / missing token | Don't print token; tell user to refresh `GITLAB_TOKEN` |
| User says "untick AC #2" | model mis-ticked | Treat as Step 4 in reverse: replace `- [x]` with `- [ ]`, push, announce |

## Principles

- **Mirror reality.** The board's state should match what the model actually did, no more, no less.
- **Announce every state change.** A silent tick or close is worse than a verbose one — the user needs to catch mistakes early.
- **Never auto-close without confirmation.** Closing is visible to teammates and feels final.
- **Never fabricate progress.** If an AC isn't verifiably met, don't tick it.
- **Don't duplicate setup logic.** The creator skill owns config setup; this one consumes it.
- **One source of truth per session.** If the model is tracking multiple issues, keep their states clearly separated in updates.
- **Re-fetch before every write.** Issue bodies can drift; race conditions cause silent overwrites.
