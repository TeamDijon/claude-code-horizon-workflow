---
name: work-epic
description: Autonomously process all children of a parent issue, or a targeted set of specific issues
disable-model-invocation: true
argument-hint: "[ISS-XXX] or [ISS-XXX ISS-YYY ...]"
---

# Work on issues $ARGUMENTS

Determine the mode based on the arguments:

- **Single argument** (e.g. `ISS-100`): This is a parent issue. Process all its children.
- **Multiple arguments** (e.g. `ISS-101 ISS-103 ISS-105`): These are specific issues. Process only these.

## Initialization

1. Check if `.claude/ticket-map.md` exists. If it does, read it.
2. If it does not exist, or if the data is stale (check the "Last updated" timestamp):
   - Fetch the parent issue from Linear using `get_issue`
   - List child issues using `list_issues` with `parentId`
   - Create or refresh the ticket map at `.claude/ticket-map.md`

### Scope check

3. **For single parent mode**: Verify the parent issue is an epic (has children) and is front-end work (belongs to the Website project or relates to theme configuration). If it is not, report to the user and stop.
4. **For targeted mode**: Each specified issue will be checked individually during Phase 1 of the processing loop.

### Build the work list

For **single parent mode**:
5. List all child issues from the ticket map.
6. Filter out issues with status Done, Canceled, or Duplicate.
7. Filter out issues already listed under "Out of scope" in the ticket map.
8. Sort remaining issues by their existing order.

For **targeted mode**:
5. Filter the ticket map (or fetch from Linear) to only the specified issue IDs.
6. Filter out issues already listed under "Out of scope" in the ticket map.

9. Present the issue list and announce that processing will begin.

## Processing loop

For each issue in the list, execute all five phases of the work-issue workflow:

### Phase 1 - Setup

Handle directly in the main conversation:
1. Fetch the issue details from Linear (or read from ticket map).
2. **Scope check**: Verify this issue is front-end work. If not, add it to the "Out of scope" section of the ticket map with a reason, report it in the progress output, and skip to the next issue.
3. Check if the parent branch exists. Create it if needed (see work-issue git-workflow.md conventions).
4. Create the issue branch from the correct base.
5. Push the branch and create a draft PR.
6. Attach the PR link to the Linear issue using `update_issue` with the `links` parameter.
7. Update Linear issue status to "In Progress".
8. Update the ticket map.

### Phase 2 - Analysis

Handle directly in the main conversation:
1. Read the issue description and comments from Linear.
2. Extract and view any images.
3. Summarize requirements, acceptance criteria, and scope.
4. In autonomous mode, proceed unless the requirements are ambiguous. If ambiguous, pause and ask the user.

### Phase 2.5 - Context check

After analysis, ask the user if there is any additional context needed before research begins (external URLs, assets, design references, content, dependencies). If the user provides context, incorporate it into the requirements. If nothing is needed, proceed immediately.

### Phase 3 - Research

**Prior knowledge:**
- If `docs/horizon-research/horizon-components.md` exists, read it and pass relevant sections to the Explore subagent(s) so they can build on existing knowledge.

**Lightweight path:**
- If the issue only involves JSON file changes and the relevant section/block types are already documented in `docs/horizon-research/horizon-components.md`, skip the full Explore agent research. Read the existing reference and go straight to formulating the implementation approach at the decision point.

**Research execution (full path):**
- Spawn 1-2 Explore agents in parallel. Provide each with:
  - The issue requirements summary from Phase 2
  - Relevant prior knowledge from the compiled reference
  - A focused search directive (e.g. one agent searches the codebase, another queries Shopify MCP docs)
- Keep subagent prompts concise. Request summaries, not raw file contents.
- Synthesize findings in the main conversation.

**Research artifact:**
- After synthesis, delegate to a general-purpose subagent to:
  - Write `docs/horizon-research/research-[ISSUE-ID].md` with components identified, approach, and gaps
  - Update `docs/horizon-research/horizon-components.md` with any newly discovered components or use cases (do not duplicate existing entries)

**Decision point:**
- **Conclusive** - Formulate the implementation approach and proceed.
- **Inconclusive** - Pause and ask the user for guidance. Do not skip to the next issue. Wait for a response.

### Phase 4 - Implementation

Delegate to a general-purpose subagent:
- Provide the subagent with:
  - The approved implementation approach
  - Specific files to modify
  - The issue ID for commit messages
  - Instructions to edit JSON files only (templates, config, locales)
  - A reminder to follow CLAUDE.md and `.claude/rules/` conventions
- The subagent commits incrementally and updates the PR description.

### Phase 5 - Validation

Delegate validation commands to a Bash subagent:
- Run `npm run check`
- Collect the output

Then in the main conversation:
- Use Shopify MCP `validate_theme` on modified files
- Check against accessibility standards
- Compare with Phase 2 acceptance criteria

Decision point:
- **Pass** - Mark PR as ready for review. Update Linear status to "In Review". Update ticket map.
- **Fail** - Fix the issues (loop back to Phase 4). If the fix is not straightforward, pause and ask the user.

### Progress report

After completing each issue (or pausing on it), report:

```
Progress: X of N issues processed

Completed:
- ISS-101: Homepage hero section - PR #XX ready for review
- ISS-102: Client logos section - PR #XX ready for review

Paused:
- ISS-103: How it works section - research inconclusive, waiting for guidance

Remaining:
- ISS-104: Product features section
```

Update the ticket map after each issue.

## Session management

After completing each issue, assess whether to continue or recommend a session split:
- If 4-5 issues have been processed in this session, recommend starting a new session
- When recommending a split: finish the current issue cleanly, update the ticket map, and tell the user to invoke `/work-epic` again in a new session - the ticket map will skip completed issues
- Keep subagent outputs to summaries only (no raw file dumps) to minimize context usage

## Completion

When all issues are processed (or the session is splitting), present a final summary:

```
Epic summary for ISS-100

Total: N issues
Completed: X (PRs ready for review)
Paused: Y (waiting for guidance)
Remaining: Z (to be processed in next session)

PRs created:
- #XX ISS-101 → parent-branch
- #XX ISS-102 → parent-branch
```
