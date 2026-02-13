---
name: work-board
description: Scan the Linear board, build a ticket map, categorize issues, and propose a work order
argument-hint: '[milestone-name] (optional)'
---

# Scan the board and plan work

Scan the Linear board for your team, build or refresh the ticket map, and propose what to work on.

If an argument is provided, use it as the milestone name. Otherwise default to the team's active milestone.

## Step 1 - Refresh the ticket map

Query Linear MCP to get the current board state:

1. Use `list_projects` to find the target project in your team.
2. Use `list_issues` with the project filter to get all issues in the target milestone.
3. For each parent issue, use `list_issues` with `parentId` to get child issues.
4. For each issue, extract: ID, title, status, parent ID, and any Linear-suggested branch name.

Write or update `.claude/ticket-map.md` with the full board state:

```markdown
# Ticket map

Last updated: [current timestamp]

## [Parent ID] - [Parent title]

Status: [status] | Branch: [branch or -] | PR: [PR number or -] → [base]

| Issue   | Title       | Status | Branch      | PR       | Base      |
| ------- | ----------- | ------ | ----------- | -------- | --------- |
| ISS-XXX | Child title | Status | branch or - | PR# or - | Parent ID |
```

## Step 2 - Scan PRs for review feedback

Use `gh` CLI commands to check for PR activity:

```bash
gh pr list --state open --json number,title,reviewDecision,comments
```

For any PR with review comments or changes requested:

```bash
gh pr view [number] --comments
```

Note which issues have PR feedback that needs action.

## Step 3 - Categorize issues

Read the ticket map and classify each issue. **First, filter by scope.**

### Scope filtering

For each issue, check eligibility:

- Must be an epic (parent with children) or a child of an in-scope epic
- Must be front-end work (belongs to the Website project, or content relates to theme configuration)

Issues that fail the scope check go to the **Out of scope** category. Add them to the "Out of scope" section at the bottom of the ticket map with a reason (e.g. "Back-end / application logic", "Not an epic", "Not front-end").

### In-scope categorization

For issues that pass the scope check:

**Ready to start**

- Status is Todo or Backlog
- No blocking dependencies
- Parent branch can be created (or already exists)

**Needs re-work**

- Status is In Review or Reviewed
- PR has comments or changes requested
- Or status was moved back from review

**In progress**

- Status is In Progress
- Work has started but is not complete

**Blocked**

- Depends on another issue that is not Done
- Requirements are unclear (flagged in previous sessions)

**Done**

- Status is Done, Canceled, or Duplicate
- No action needed

**Out of scope**

- Not an epic or child of an epic
- Not front-end / theme work
- Recorded in ticket map with reason, never re-evaluated

## Step 4 - Present the assessment

Display a board summary:

```
Board summary for "[milestone name]"
Last updated: [timestamp]

Ready to start: X issues
  - ISS-101: Homepage hero section
  - ISS-102: Client logos section

Needs re-work: Y issues
  - ISS-103: How it works section (2 PR comments)

In progress: Z issues
  - ISS-104: Product features section

Blocked: W issues
  - (none)

Done: V issues
  - ISS-105: Navigation setup

Out of scope: U issues
  - ISS-200: API rate limiting (back-end / application logic)
```

## Step 5 - Propose a work order

Based on the categorization:

1. **Re-work first** - Issues with PR feedback should be addressed before starting new work.
2. **Then new work** - Ready-to-start issues, grouped by parent for efficient branch management.
3. **Skip blocked** - Flag blocked issues but do not plan work for them.

Present the proposed order and estimated scope (number of issues to process).

Ask the user to confirm or adjust the order.

## Step 6 - Execute

On user approval, begin processing the issues using the work-epic workflow inline:

- For re-work items: read the PR comments, analyze what needs to change, implement fixes, re-validate.
- For new items: execute the full five-phase workflow (setup, analysis, research, implementation, validation).
- Follow the same session management rules (recommend splitting after 4-5 issues).

Update the ticket map after each issue.
