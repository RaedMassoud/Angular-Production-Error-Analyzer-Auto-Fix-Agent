# Biweekly Angular Production Error Analyzer & Auto-Fix Agent

A Claude Code cloud routine that automatically triages unresolved Sentry errors in a production Angular app, performs deep root-cause analysis, and - for clear, contained bugs - creates the Jira ticket, opens a draft fix PR on GitHub, and posts a digest to Slack. All without a human in the loop.

---

## What It Does

Every two weeks, the agent wakes up, pulls the top unresolved production errors from Sentry, scores them by impact, and works through each one:

- If it can fix it confidently: creates a Jira bug ticket, branches off master, applies the fix, commits, opens a draft PR, and marks the Sentry issue as resolved.
- If the root cause is in the backend or the fix is ambiguous: documents the analysis clearly and flags it for human review.
- Either way: posts a full digest to a Slack channel so the team knows exactly what happened and why.

The result is a queue of draft PRs and pre-filled Jira tickets waiting for human review each sprint - not hours of manual triage.

---

## How It Works - Step by Step

### Step 0 - Biweekly Gate
The routine is scheduled weekly but runs on odd ISO weeks only. It checks `date +%V` and exits immediately on even weeks. This gives the team time to review and merge fixes before the next cycle picks up new issues.

### Step 1 - Fetch Issues
Pulls the 10 most recent unresolved issues from Sentry (production environment only). Reads summary data only at this stage - no heavy event fetching yet - to keep the session lean.

### Step 2 - Qualify
Filters out noise before scoring:
- Issues last seen more than 30 days ago are skipped (likely already fixed, just not marked resolved).
- Issues with fewer than 3 events are skipped (too rare to be actionable).
- Already resolved or ignored issues are skipped.
- Issues that are old but still actively firing are flagged as "chronic" and highlighted for human attention.

### Step 2b - Confidence Rating
Each remaining issue is rated HIGH or LOW confidence before scoring:
- **HIGH**: null/undefined access at a specific component, missing null check, isolated type error - something with a clear, contained fix touching 1-2 files.
- **LOW**: auth flows, interceptors, app initializer, backend-dependent issues, cascade errors, or anything where the root cause is ambiguous from the event data alone.

LOW-confidence issues are deprioritized in the top 3 if HIGH-confidence alternatives exist. The agent is designed to clear simple bugs, not guess at hard ones.

### Step 3 - Score and Rank
Issues are scored by:

```
score = (userCount / max_userCount) + (eventCount / max_eventCount)
```

Ties are broken by userCount. Top 3 proceed; LOW-confidence issues are swapped out for HIGH-confidence ones from the remaining pool if available.

### Step 4 - Check for Existing Jira Tickets
For each of the top 3 issues, the agent checks Sentry's external-issues API to see if a Jira ticket is already linked. If yes, it skips that issue entirely and notes it in the Slack digest - no duplicate work.

### Step 5 - Fetch Event Details
Pulls the single most recent Sentry event per issue. Extracts: stack trace, route, HTTP method, request parameters, user context, browser/device tags, and breadcrumbs.

### Step 6 - Deep Root-Cause Analysis
This is the core of the routine. For each issue, the agent:

1. Identifies the exact file and line number from the stack trace.
2. Reads the full file in the repository - not just the failing line - along with related components, services, and templates.
3. Asks a key question before suggesting any fix: *"Should this data state be possible at all?"* If no, it traces the value upstream rather than adding a frontend null-check that silences a backend bug.
4. Derives reproduction steps from the Sentry event data (route, actions, context).
5. Classifies the issue:
   - **CODE BUG** - clear defect, contained fix exists. Proceed to fix.
   - **FRONTEND PARTIAL FIX** - safe frontend improvement exists but root cause is upstream. Apply the fix, document the backend concern.
   - **BACKEND/API ISSUE** - Angular code is correct, backend sends bad data. Document and flag.
   - **BUSINESS LOGIC CONCERN** - behavior is wrong at a higher level. Document and flag.
   - **UNCERTAIN** - root cause is ambiguous, no safe partial fix exists. Flag with analysis, do not guess.

### Step 7 - Fix, Ticket, PR (CODE BUG and FRONTEND PARTIAL FIX only)

The agent follows a strict order to ensure everything links together automatically in Jira and GitHub:

1. **Find the active sprint** - queries Jira to get the current sprint ID so the ticket lands in the right sprint.
2. **Create the Jira bug ticket** - includes root cause analysis, reproduction steps, and a link to the Sentry issue. Sets the team and moves the ticket to the ready status.
3. **Link Jira ticket to Sentry** - attempts to register the Jira ticket as an external issue on the Sentry issue, so future runs by other team members will detect it and skip.
4. **Create the fix branch** - named using the Jira issue key (e.g. `PD-1234-fix-dashboard-null-ref`) so GitHub's Jira integration auto-links the branch, commits, and PR to the ticket.
5. **Apply the fix** - minimal changes only, following existing code style.
6. **Commit** - includes the Jira issue key in the commit message for automatic linking.
7. **Open a draft PR** - includes root cause, reproduction steps, what was fixed, and links to both the Sentry issue and Jira ticket.
8. **Mark the Sentry issue as resolved** - this is the primary deduplication gate. Once resolved, `is:unresolved` in Step 1 won't return it in future runs by any team member.

### Step 8 - Slack Digest
Posts one message to the team's Slack channel covering all 3 issues:
- Error title, Sentry link, users affected, event count, age
- Root cause summary
- Reproduction steps (condensed)
- Status (PR created / flagged for human review / backend issue / etc.)
- Links to the PR and Jira ticket if created

---

## Multi-User Deduplication

The routine is safe for multiple team members to run independently. Two mechanisms prevent duplicate work:

1. **Step 4 (Jira link check)** - if a Jira ticket is already linked to a Sentry issue from a previous run, it is skipped.
2. **Step 7g (mark as resolved)** - after creating a PR, the Sentry issue is marked resolved. Subsequent runs won't see it in `is:unresolved` queries at all.

This means whoever runs the routine first claims those issues, and the next person's run naturally surfaces a fresh set.

---

## Requirements

| Service | What it's used for |
|---|---|
| **Sentry** | Fetch issues, fetch events, mark as resolved |
| **Jira (Atlassian)** | Create bug tickets, set team, move to sprint |
| **GitHub** | Create branches, commit, open draft PRs |
| **Slack** | Post the biweekly digest |
| **Claude Code Routines** | Cloud execution environment |

The routine runs as a Claude Code cloud agent - it has its own isolated git checkout, runs in Anthropic's cloud infrastructure, and requires no local machine or CI pipeline.

---

## Configuration

To adapt this routine for your own project, replace the following placeholders in the prompt:

| Placeholder | What it is |
|---|---|
| `<YOUR_APP_NAME>` | Your app's display name |
| `<YOUR_GITHUB_ORG>/<YOUR_REPO>` | GitHub repository path |
| `<YOUR_SENTRY_ORG_SLUG>` | Sentry organization slug (from your Sentry URL) |
| `<YOUR_SENTRY_PROJECT>` | Sentry project name |
| `<YOUR_ATLASSIAN_CLOUD_ID>` | Atlassian site ID - call `getAccessibleAtlassianResources` via the Atlassian MCP |
| `<YOUR_JIRA_PROJECT_KEY>` | Jira project key (e.g. PD, ENG, FE) |
| `<YOUR_ATLASSIAN_ACCOUNT_ID>` | Your Atlassian account ID - call `atlassianUserInfo` via the Atlassian MCP |
| `<YOUR_ATLASSIAN_DOMAIN>` | Your Atlassian domain (the `xxx` in `xxx.atlassian.net`) |
| `<YOUR_TEAM_NAME>` | Team name in Jira |
| `<YOUR_TEAM_FIELD_VALUE>` | UUID of the team in Jira's custom field |
| `<YOUR_READY_FOR_SPRINT_STATUS>` | Name of your Jira "ready" column status |
| `<YOUR_SLACK_CHANNEL>` | Slack channel to post digests to |

---

## Design Principles

- **Fix at the right layer.** Before adding any guard or null-check, the agent asks whether the bad data state should be possible at all. If not, the root cause is upstream - adding a frontend patch that silences a backend bug is worse than no fix.
- **Minimal blast radius.** Only the directly broken code is touched. No refactoring, no cleanup, no unrelated changes.
- **Draft PRs only.** The agent never merges or pushes directly to master. Every fix goes through human review.
- **A wrong fix is worse than no fix.** When confidence is LOW or the root cause is ambiguous, the agent flags and documents rather than guessing.
- **Claim issues before working on them.** Marking as resolved immediately after creating the PR ensures no other agent run (or team member's run) picks up the same issue.
