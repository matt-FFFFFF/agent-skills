---
name: pr-flow
description: End-to-end pull-request workflow. Creates a PR with a Conventional-Commits title and a structured body, waits for CI to finish, fixes test failures in a loop, requests a GitHub Copilot review, addresses review comments (commits fixes, posts inline replies, resolves threads), and re-requests Copilot review until the PR is clean. Use this whenever the user asks to "submit", "open a PR", "ship", "create and merge", or otherwise drive a change through review.
license: MIT
metadata:
  audience: maintainers
  workflow: github
---

# pr-flow

End-to-end PR creation + iteration. Stops only when CI is green AND
Copilot review returns no actionable comments (or the user tells you
to stop).

Assumes `gh` is authenticated and the current branch already has the
intended commits. If the working tree is dirty, refuse and ask the
user to commit/stash first.

---

## 0. Pre-flight

```bash
git status --porcelain          # must be empty
git rev-parse --abbrev-ref HEAD # must NOT be main / master
gh auth status                  # must be logged in
```

If any check fails, stop and ask.

Determine the base branch:

```bash
gh repo view --json defaultBranchRef -q .defaultBranchRef.name
```

Read recent history for context — title style, scope conventions,
whether the repo uses scopes, etc.:

```bash
git log --oneline -20
git log "$BASE..HEAD" --format='%h %s%n%b' # commits in this PR
```

---

## 1. Create the PR

### Title — Conventional Commits

`<type>(<scope>): <imperative summary, no trailing period>`

- `type` ∈ `feat | fix | refactor | perf | docs | test | build | ci | chore | revert`
- `scope` is optional; use it when the repo's recent history does
  (e.g. `feat(api):`, `fix(loader):`). Match the repo's prevailing
  scope vocabulary — read `git log` first.
- Summary ≤ 72 chars, imperative mood ("add", not "added"), no period.
- `!` after type/scope (or `BREAKING CHANGE:` footer) marks breaking.

If the branch is a single commit and that commit's subject is already
a clean Conventional-Commits line, reuse it verbatim. If multiple
commits, pick the type/scope that describes the **net** change.

### Body — required sections

```markdown
## Summary

<1-3 bullets describing the WHY and the user-visible effect>

## Changes

<bullet per file group / module — what changed and why>

## Verification

<commands run + their outcome: build/lint/test/typecheck/etc.>

## Notes

<deferred work, follow-ups, deviations from spec, breaking changes>
```

Drop sections that don't apply (e.g. "Notes" if there are none).
Reference issues with `Closes #N` / `Refs #N` in a footer when
relevant.

### Open it

```bash
git push -u origin "$(git rev-parse --abbrev-ref HEAD)"
gh pr create --base "$BASE" --title "$TITLE" --body "$(cat <<'EOF'
... body ...
EOF
)"
```

Capture the PR URL and number. Everything below uses `$PR`.

---

## 2. CI loop — wait, diagnose, fix, repeat

```bash
gh pr checks "$PR" --watch --interval 15 || true
gh pr checks "$PR" --json name,state,conclusion,detailsUrl
```

Classify each non-success check:

- `IN_PROGRESS` / `QUEUED` — keep watching.
- `SUCCESS` / `NEUTRAL` / `SKIPPED` — accept.
- `FAILURE` / `CANCELLED` / `TIMED_OUT` / `ACTION_REQUIRED` — fix.

For every failed check:

```bash
RUN_ID=$(gh run list --branch "$(git rev-parse --abbrev-ref HEAD)" \
  --workflow "<workflow file or name>" --limit 1 --json databaseId \
  -q '.[0].databaseId')
gh run view "$RUN_ID" --log-failed | tail -200
```

Categorise:

| Symptom                                  | Action                                  |
| ---------------------------------------- | --------------------------------------- |
| Linter / formatter / type error          | Fix in code, commit, push.              |
| Flaky / infra failure (network, runner)  | `gh run rerun "$RUN_ID" --failed` once. |
| Real test failure                        | Reproduce locally, fix, commit, push.   |
| Required check blocked on missing secret | Stop; surface to user.                  |

Each fix gets its own commit:

```
fix(<scope>): <what + why, e.g. "satisfy tflint terraform_unused_declarations">
```

**Never** rewrite history on a PR branch unless the user explicitly
asks (no `--amend`, no `--force-with-lease`). Stack new fix commits.

After every push, loop back to `gh pr checks --watch`. Bail out of
the loop only when:

- All required checks succeed, OR
- The same failure has occurred 3 times after 3 different fix
  attempts — stop and ask the user.

---

## 3. Request Copilot review

```bash
gh api -X POST .../requested_reviewers \
  -f 'reviewers[]=copilot-pull-request-reviewer[bot]'
```

Reviewer-name confusion (all three strings refer to the same actor
in different surfaces — use the one each surface expects):

| Surface                           | Login string                         |
| --------------------------------- | ------------------------------------ |
| Display login on submitted review | `copilot-pull-request-reviewer[bot]` |
| GraphQL `botLogins` argument      | `copilot-pull-request-reviewer`      |
| `requested_reviewers[].login`     | `Copilot`                            |
| `gh pr create --reviewer ...`     | `Copilot` (works on create only)     |

---

## 4. Wait for Copilot's review

Poll until the review is submitted (Copilot usually finishes in 1-3
minutes; cap at 10):

```bash
# NOTE: filtering uses the SUBMITTED-review login, which is the bot
# user with the [bot] suffix — NOT the "Copilot" alias used to request.
COPILOT_BOT_LOGIN='copilot-pull-request-reviewer[bot]'

for i in $(seq 1 40); do
  state=$(gh api "repos/$OWNER/$REPO/pulls/$NUMBER/reviews" \
    --jq --arg u "$COPILOT_BOT_LOGIN" \
      '[.[] | select(.user.login==$u)][-1].state // "PENDING"')
  [[ "$state" != "PENDING" ]] && break
  sleep 15
done
```

Fetch comments + threads:

```bash
# Inline review comments (filter to Copilot)
gh api "repos/$OWNER/$REPO/pulls/$NUMBER/comments" \
  --jq --arg u "$COPILOT_BOT_LOGIN" \
    '.[] | select(.user.login==$u) | {id, path, line, body, in_reply_to_id}'

# Top-level review summary
gh api "repos/$OWNER/$REPO/pulls/$NUMBER/reviews" \
  --jq --arg u "$COPILOT_BOT_LOGIN" \
    '.[] | select(.user.login==$u) | {id, state, body}'

# Thread IDs (needed for resolve)
gh api graphql -f query='
query($owner:String!,$repo:String!,$num:Int!){
  repository(owner:$owner,name:$repo){
    pullRequest(number:$num){
      reviewThreads(first:100){
        nodes{ id isResolved comments(first:10){ nodes{ id databaseId body author{login} path line }}}
      }
    }
  }
}' -F owner="$OWNER" -F repo="$REPO" -F num="$NUMBER"
```

---

## 5. Triage and act

For each unresolved Copilot thread, decide:

| Verdict      | Action                                                                                   |
| ------------ | ---------------------------------------------------------------------------------------- |
| **Accept**   | Make the change. Commit. Reply explaining what was done. Resolve thread.                 |
| **Reject**   | Do not change code. Reply with concrete reason (factual, not defensive). Resolve thread. |
| **Defer**    | Open a follow-up issue. Reply linking to it. Resolve thread.                             |
| **Question** | Reply asking for clarification. Do **not** resolve. Surface to user if it blocks merge.  |

Be honest: if Copilot is wrong, say so and explain why (cite line
numbers, prior decisions, PLAN/ADR references). If it's right, fix
it without hedging.

### Commit fixes

Group by topic, one commit per logical change:

```
fix(<scope>): <what>

Addresses Copilot review thread on <file>:<line>.
```

Push after each commit (or batch then push — either is fine).

### Reply + resolve via GraphQL

```bash
# Reply to a thread (creates a child review comment)
gh api graphql -f query='
mutation($threadId:ID!,$body:String!){
  addPullRequestReviewThreadReply(input:{pullRequestReviewThreadId:$threadId,body:$body}){
    comment{ id url }
  }
}' -F threadId="$THREAD_ID" -F body="$REPLY"

# Resolve
gh api graphql -f query='
mutation($threadId:ID!){
  resolveReviewThread(input:{threadId:$threadId}){ thread{ id isResolved } }
}' -F threadId="$THREAD_ID"
```

If `addPullRequestReviewThreadReply` returns EOF / 502, retry up to
3× with backoff before surfacing.

---

## 6. Re-request Copilot

After all threads handled and fixes pushed:

```bash
# Wait for the new commit's CI to go green first (loop §2 again).
# Then re-request
gh api -X POST .../requested_reviewers \
  -f 'reviewers[]=copilot-pull-request-reviewer[bot]'
```

```bash
# Wait for the new commit's CI to go green first (loop §2 again).
# Then re-request — MUST use the GraphQL botLogins path from §3,
# the REST POST silently no-ops once Copilot has reviewed.
gh api graphql -f query='
mutation($prId:ID!,$logins:[String!]){
  requestReviewsByLogin(input:{
    pullRequestId:$prId,botLogins:$logins,union:true
  }){ pullRequest{ reviewRequests(first:10){ nodes{
    requestedReviewer{ __typename ... on Bot{login} }
  }}}}
}' -F prId="$PR_ID" -f 'logins[]=copilot-pull-request-reviewer'

# Verify a new review_requested event landed:
gh api "repos/$OWNER/$REPO/issues/$NUMBER/timeline" \
  --jq '[.[] | select(.event=="review_requested")] | last'
```

Loop back to §4. Exit conditions:

- New review is `APPROVED` or has no actionable comments (only
  praise / nits already dismissed) → done.
- Same comment re-emerges after a fix the bot keeps rejecting → stop
  and surface to the user; the bot is in a loop and a human has to
  break it.
- 5 review rounds elapsed → stop and surface.

---

## 7. Hand back to the user

When done, report:

- PR URL.
- Final commit SHA.
- CI status.
- Round count + summary of Copilot threads (accepted / rejected /
  deferred).
- Any open questions or deferred follow-ups.

Do **not** merge. Merging is the user's call unless they explicitly
delegated it.

---

## Hard rules

- Never `git push --force` or `--force-with-lease` on a PR branch
  unless the user explicitly says so.
- Never `git commit --amend` after a commit has been pushed.
- Never bypass branch protection (`--admin`, `--no-verify`).
- Never resolve a thread you haven't replied to.
- Never claim a fix landed without verifying the new commit is on
  origin and CI re-ran.
- If a check requires a secret you don't have access to, stop and
  ask — don't paper over it.

## Repo conventions to honour

Before opening the PR, check for and respect:

- `CONTRIBUTING.md` / `AGENTS.md` / `CLAUDE.md` — workflow rules.
- `.github/PULL_REQUEST_TEMPLATE.md` — body skeleton.
- `commitlint.config.*` / `.commitlintrc.*` — strict commit format.
- `CODEOWNERS` — additional reviewers may be auto-added; don't
  remove them.
- Repo-side STATUS / changelog files that may need updating in the
  same PR (e.g. this repo's `STATUS.md`).
