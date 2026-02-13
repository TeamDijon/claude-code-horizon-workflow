---
name: work-issue
description: Work on a single Linear issue through setup, analysis, research, implementation, and validation phases
disable-model-invocation: true
argument-hint: "[ISS-XXX]"
---

# Work on issue $ARGUMENTS

Execute the five-phase workflow for this Linear issue. Follow each phase in order. Do not skip phases.

## Phase 0 - Scope check

Before any work, verify this issue is in scope.

1. Fetch the issue from Linear using MCP (`get_issue` with the issue identifier).
2. Check eligibility:
   - The issue must be an **epic** (has children), OR a **child of an in-scope epic**
   - The issue must be **front-end work** (belongs to the Website project, or its content clearly relates to front-end/theme configuration)
3. If the issue is **out of scope**:
   - Report to the user: "Issue [ID] - [Title] is out of scope. Reason: [back-end / not an epic / not front-end]"
   - If `.claude/ticket-map.md` exists, add the issue to the "Out of scope" section with the reason
   - **Stop. Do not proceed to Phase 1.**
4. If `.claude/ticket-map.md` exists, check if this issue is already listed under "Out of scope". If so, report it and stop.

## Phase 1 - Setup

1. If `.claude/ticket-map.md` exists, read it for context.
2. Extract the issue title, description, parent issue (if any), and the **git branch name** from Linear.
3. Determine the branch base:
   - If this issue has a parent issue, it is a child. The base branch is the parent's branch.
   - If this issue has no parent, it is a parent. The base branch is `main`.
4. **If this is a child issue, verify the parent branch exists locally and on the remote.** If it does not:
   - Fetch the parent issue from Linear to get its branch name
   - Create the parent branch from `main`
   - Push it to the remote
   - Create a draft PR for the parent (targeting `main`)
5. Create the issue branch using the Linear-suggested branch name, based off the correct base branch.
6. Push the branch to the remote.
7. Create a draft PR:
   - Child issue PR targets the parent branch
   - Parent issue PR targets `main`
   - PR title matches the issue title
   - PR body references the Linear issue ID
8. Attach the PR link to the Linear issue using `update_issue` with the `links` parameter (URL and title of the PR).
9. Update the Linear issue status to "In Progress".
10. If `.claude/ticket-map.md` exists, update the row for this issue with the branch name and PR number.

See [git-workflow.md](git-workflow.md) for detailed conventions.

## Phase 2 - Analysis

1. Read the full issue description from Linear.
2. Read all comments on the issue using `list_comments`.
3. If the description contains images, use `extract_images` to view them.
4. Summarize:
   - What needs to be done (requirements)
   - How to verify it is done (acceptance criteria)
   - Scope boundaries (what is and is not included)
5. Present the summary to the user and wait for confirmation before proceeding.

## Phase 2.5 - Context check

After the user confirms the analysis, ask if there is any additional context needed before research begins. Examples of what might be needed:

- External URLs (app store links, documentation pages, third-party services)
- Specific assets (images, logos, icons) not yet uploaded
- Design references or mockups
- Content that needs to be written or provided
- Dependencies on other issues or systems

If the user provides context, incorporate it into the requirements before proceeding to Phase 3. If they have nothing to add, proceed immediately.

## Phase 3 - Research

### Prior knowledge

1. If `docs/horizon-research/horizon-components.md` exists, read it. This is the compiled reference of all prior Horizon research.
2. Pass relevant sections of this reference to the Explore subagent(s) as prior knowledge so they can skip already-known components and focus on gaps.

### Lightweight path

3. Check if the issue only involves JSON file changes (templates, config, locales) and the relevant section/block types are already documented in `docs/horizon-research/horizon-components.md`. If so, skip the full Explore agent research - read the existing reference and go straight to formulating the implementation approach at the decision point (step 6).

### Research execution (full path)

4. Spawn Explore subagent(s) to:
   - Search the codebase for existing sections, blocks, templates, and config that are relevant to this issue
   - Use the Shopify MCP tools (`search_docs_chunks`, `fetch_full_docs`, `learn_shopify_api`) to look up Horizon theme capabilities that match the requirements
   - Build on prior knowledge rather than re-discovering known components
5. Synthesize the findings:
   - Which Horizon components match the requirements
   - What configuration options are available (settings, blocks, presets)
   - What gaps exist (requirements that Horizon does not cover natively)

### Research artifact

6. After synthesis, delegate to a general-purpose subagent to:
   - Create `docs/horizon-research/` directory if it does not exist
   - Write a per-issue research file at `docs/horizon-research/research-[ISSUE-ID].md` with:
     - Horizon components identified (section type, purpose, key settings, blocks, what it is useful for)
     - Configuration approach recommended for this issue
     - Gaps found (things Horizon does not provide natively)
   - Update `docs/horizon-research/horizon-components.md` by merging any newly discovered components, settings, or use cases. Add a "Used in: [ISSUE-ID]" reference. Do not duplicate entries that already exist - only add new information.

### Decision point

7. Decision point:
   - **If research is conclusive** - Present the implementation approach (which JSON files to modify, what sections/blocks to add/configure) and wait for user approval.
   - **If research is inconclusive** - Present what was found, explain the gaps clearly, and ask the user for guidance. Do NOT proceed to implementation.

Important: This milestone is configuration only. Research looks for existing Horizon components to configure through JSON. Do not propose Liquid, CSS, or JS edits.

## Phase 4 - Implementation

Only proceed if Phase 3 produced a clear, user-approved approach.

1. Delegate implementation to a general-purpose subagent. Provide it with:
   - The approved implementation approach from Phase 3
   - The specific files to modify
   - The issue ID for commit messages
2. The subagent should:
   - Add or update JSON files only (templates, config, locales)
   - Commit incrementally with descriptive messages referencing the issue ID (e.g. "feat(ISS-101): add hero section to homepage template")
   - Follow the conventions in CLAUDE.md and `.claude/rules/`
3. After implementation, update the PR description with a summary of changes made.

## Phase 5 - Validation

1. Run `npm run check` (Shopify theme linter) via a Bash subagent.
2. Use the Shopify MCP `validate_theme` tool on all modified files.
3. Review changes against the acceptance criteria from Phase 2.
4. Check modified files against relevant accessibility standards in `.claude/rules/*-accessibility.md`.
5. Compile a validation report.
6. Decision point:
   - **If validation passes** - Mark the PR as ready for review. Update the Linear issue status to "In Review". Update the ticket map if it exists. Report success to the user.
   - **If validation fails** - List the specific failures. Loop back to Phase 4 to fix them.

After "In Review", the human takes over for visual and functional verification.

See [validation-checklist.md](validation-checklist.md) for the full checklist.
