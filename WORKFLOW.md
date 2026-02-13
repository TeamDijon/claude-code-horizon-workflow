# Agentic workflow for Horizon Shopify theme projects

## Context

The project uses the Horizon Shopify theme and tracks work in Linear. Issues follow a parent/child pattern (e.g. ISS-100 "Homepage" has children ISS-101, ISS-102, etc.).

The scope of this milestone is configuration only - finding and configuring existing Horizon sections, blocks, and templates through JSON files. No Liquid/CSS/JS edits. A future milestone will cover custom development.

The goal is to create a repeatable, skill-driven workflow that processes issues through setup, analysis, research, implementation, and validation, then scales to batch-process multiple issues autonomously.

---

## How subagents work (technical context)

A subagent is launched via the Task tool. It gets its own isolated context window, runs independently, and returns a summary to the main conversation. This keeps the main context lean.

**Constraints:**

- Subagents cannot spawn other subagents (flat hierarchy, one level deep)
- Subagents cannot invoke skills
- The main conversation is always the orchestrator

**Available subagent types:**

- **Explore** - Read-only. Can use Glob, Grep, Read, and MCP tools (Shopify MCP, Linear MCP). Good for research. Cannot edit files.
- **general-purpose** - Full access. Can read, write, edit, use Bash, use MCP tools. Good for implementation sub-tasks.
- **Bash** - Command execution only. Good for running linters, git operations.

**Orchestration pattern for `/work-epic`:**

```
Main conversation (loaded with /work-epic skill prompt)
│
├── Reads ticket map (local markdown file, avoids MCP overhead)
│
├── For issue ISS-101:
│   ├── Explore agent → research codebase + Shopify docs
│   ├── Main synthesizes findings, decides approach
│   ├── general-purpose agent → implement JSON changes + commit
│   └── Bash agent → run validation (npm run check)
│
├── For issue ISS-102:
│   ├── Explore agent → research
│   ├── Main synthesizes
│   ├── general-purpose agent → implement
│   └── Bash agent → validate
│
└── Summary of all work
```

The main conversation stays lean because heavy work (searching files, reading docs, editing files) happens inside subagents. Their full output stays in their own context; only the summary comes back.

**Session management for long batches:**
The ticket map is the key to session continuity. Because it records the status, branch, and PR for every issue, a new session can pick up exactly where the previous one left off.

Guidance built into the skill prompts:

- After completing each issue, the orchestrator checks context usage (rough heuristic: if more than 4-5 issues have been processed in a single session, recommend starting fresh)
- When recommending a session split, the skill instructs Claude to finish the current issue cleanly, update the ticket map, and tell the user to start a new session with the same `/work-epic` command - the ticket map will skip already-completed issues
- For compaction: keep subagent prompts focused and concise, avoid returning raw file contents from subagents (summaries only), and let auto-compression handle the rest

---

## Ticket map

A local markdown file (`.claude/ticket-map.md`) that caches the Linear board state. This avoids repeated MCP queries that consume context tokens.

**Structure:**

```markdown
# Ticket map

Last updated: 2026-02-12T14:30:00

## ISS-100 - Homepage

Status: In Progress | Branch: feature/ISS-100-homepage | PR: #12 → main

| Issue   | Title                 | Status | Branch | PR  | Base    |
| ------- | --------------------- | ------ | ------ | --- | ------- |
| ISS-101 | Homepage hero section | Todo   | -      | -   | ISS-100 |
| ISS-102 | Client logos section  | Todo   | -      | -   | ISS-100 |
| ISS-103 | How it works section  | Todo   | -      | -   | ISS-100 |

## Out of scope

| Issue   | Title               | Reason                       |
| ------- | ------------------- | ---------------------------- |
| ISS-200 | API rate limiting   | Back-end / application logic |
| ISS-201 | Database migrations | Back-end / application logic |
```

**Location:** `.claude/ticket-map.md` - lives alongside other Claude working files. Add to `.gitignore`.

**Maintained by the orchestrator:**

- `/work-board` creates and refreshes the full map from Linear MCP (one-time query)
- `/work-epic` reads the map at start, updates entries as issues progress, skips issues already marked Done or out of scope
- `/work-issue` updates its own row on completion, or adds to "Out of scope" if ineligible
- The map is a working document, not committed to git
- Serves as session continuity - new sessions read it to know where to resume

---

## Scope filtering

All three skills enforce an eligibility check. An issue is in scope only if:

- It is an epic (parent issue with children), OR a child of an in-scope epic
- AND it is front-end work (belongs to the Website project, or relates to theme configuration)

Issues that fail the check are marked as "out of scope" in the ticket map with a reason and are never re-evaluated.

---

## Research knowledge accumulation

Every research phase produces a structured markdown file in `docs/horizon-research/` (gitignored). Two types of files:

- **Per-issue files** (`research-ISS-XXX.md`) - Components identified, configuration approach, and gaps for a specific issue
- **Compiled reference** (`horizon-components.md`) - Single file updated after each session, merging all unique findings into one searchable reference

Future research agents receive the compiled reference as prior knowledge, so they can skip already-known components and focus on gaps. Over time, this can be promoted to a `.claude/rules/` file.

---

## Skill 1 - `/work-issue [ISS-XXX]`

The core workflow skill. Takes a single issue ID and walks through six phases (including scope check).

### Phase 0 - Scope check

- Fetch the issue from Linear
- Verify it is an epic or child of an in-scope epic, and is front-end work
- If out of scope, record in ticket map and stop

### Phase 1 - Setup

- Fetch the issue from Linear (via MCP or ticket map if available) to get title, description, parent info, and **git branch name**
- **If this is a child issue, check that the parent branch exists**
  - If the parent branch does not exist, create it first (fetch parent issue, create branch from main, create draft PR for parent)
  - Then create the child branch from the parent branch
- If this is a parent issue, create branch from `main`
- Use the **Linear-suggested branch name**
- Create a draft PR targeting the correct base (child → parent branch, parent → main)
- Attach the PR link to the Linear issue using `update_issue` with the `links` parameter
- Update Linear issue status to "In Progress"
- Update ticket map if it exists

### Phase 2 - Analysis

- Read the full issue description and any comments from Linear
- If the issue has attachments/images, extract and view them
- Summarize the requirements, acceptance criteria, and scope
- Present the analysis to the user for confirmation before proceeding

### Phase 2.5 - Context check

- After the user confirms the analysis, ask if there is any additional context needed before research begins (external URLs, assets, design references, content, dependencies)
- If context is provided, incorporate it into the requirements before proceeding

### Phase 3 - Research

- Read `docs/horizon-research/horizon-components.md` (if it exists) and pass relevant sections to research subagents as prior knowledge
- **Lightweight path** - If the issue only involves JSON file changes and the relevant section/block types are already documented in the compiled reference, skip the full Explore agent research and go straight to proposing the implementation approach
- **Full path** - Spawn Explore subagent(s) to
  - Search the codebase for relevant existing sections, blocks, templates, and config
  - Use Shopify MCP tools to look up Horizon theme documentation and capabilities
  - Build on prior knowledge rather than re-discovering known components
- Synthesize findings in the main conversation
  - What Horizon components match the requirements
  - What configuration options are available
  - What gaps exist (things the ticket asks for that Horizon does not provide natively)
- Write research artifact: per-issue file to `docs/horizon-research/research-ISS-XXX.md`, update compiled reference in `docs/horizon-research/horizon-components.md`
- **If research is conclusive** → present implementation approach for approval
- **If research is inconclusive** → present what was found, explain the gaps, and ask for guidance. Do not proceed to implementation.

### Phase 4 - Implementation

Only reached if research produced a clear, approved approach.

- Add/update JSON files only (templates, config, locales)
- Commit incrementally with descriptive messages referencing the issue ID
- Update the draft PR description with a summary of changes

### Phase 5 - Validation

- Run `npm run check` (Shopify theme linter)
- Use Shopify MCP `validate_theme` tool on modified files
- Run accessibility audit against relevant `.claude/rules/*-accessibility.md` standards
- Review changes against the analysis from Phase 2
- If validation passes → mark PR as ready for review, update Linear status to "In Review", update ticket map
- If validation fails → loop back to Phase 4 with specific issues to fix

After "In Review", the human takes over for visual/functional verification.

---

## Skill 2 - `/work-epic [ISS-XXX]` or `/work-epic [ISS-XXX ISS-YYY ...]`

Autonomous batch orchestrator.

**Single parent mode** (`/work-epic ISS-100`):

1. Read ticket map (or fetch from Linear if no map exists, then create one)
2. Verify the parent issue is an in-scope epic. If not, stop.
3. List all child issues, filter out Done/Canceled/Duplicate and out-of-scope issues
4. For each child, run the scope check during setup. Out-of-scope children are recorded and skipped.
5. Process each eligible child sequentially using the `/work-issue` workflow phases inline
6. Delegate research to Explore subagents, implementation to general-purpose subagents, validation to Bash subagents
7. Only pause if research is inconclusive, a decision is needed, or validation fails
8. After each child, update ticket map and report progress (X of N)
9. When done, summarize all PRs and statuses

**Targeted mode** (`/work-epic ISS-101 ISS-104 ISS-105`):

- Process only the specified issues
- Useful for re-work after review comments or picking up specific incomplete work

**How subagents are used per issue:**

- 1-2 Explore agents (parallel) for Phase 3 research
- 1 general-purpose agent for Phase 4 implementation (receives the research summary and approved approach as input)
- 1 Bash agent for Phase 5 validation commands
- Main conversation handles Phase 1 (setup), Phase 2 (analysis), synthesis between phases, and decisions

---

## Skill 3 - `/work-board`

Board-level agent that scans the current state and decides what to do.

**Specialized tools to reduce MCP overhead:**

1. Query Linear MCP once to build/refresh the ticket map
2. All subsequent reasoning works from the local ticket map file
3. For PR comment checks, use `gh` CLI commands (`gh pr list`, `gh pr view --comments`) instead of additional MCP calls

**Flow:**

1. Refresh ticket map from Linear MCP (single batch query)
2. Scan PRs via `gh` CLI for review comments or requested changes
3. Categorize issues (scope check first - non-epic and non-front-end issues go to "Out of scope")
   - Ready to start (Todo, no blockers)
   - Needs re-work (In Review with PR comments, or sent back)
   - Blocked (waiting on dependencies)
   - Done (no action needed)
   - Out of scope (not an epic, not front-end, recorded with reason)
4. Present the assessment and proposed work order
5. On user approval, begin working through issues using the `/work-epic` workflow inline

---

## Key design decisions

1. **Configuration only** - This milestone is purely about finding and using existing Horizon components through JSON. No Liquid/CSS/JS edits. Research identifies what Horizon offers; implementation configures it.

2. **Research can stop the workflow** - If research does not find a clear path, the agent presents findings and asks for guidance rather than forcing an implementation.

3. **Parent branch prerequisite** - Before working on a child issue, the agent verifies the parent branch exists and creates it if needed.

4. **Ticket map as local cache** - A markdown file caches the board state to avoid repeated MCP queries. Updated incrementally as work progresses.

5. **Flat subagent hierarchy** - Main conversation orchestrates. Subagents do focused work (research, implementation, validation) and return summaries. No nesting.

6. **Linear branch names** - Branch names come from Linear's suggested git branch name.

7. **Human handoff at "In Review"** - Agent's job ends when the PR is ready for review. Human handles visual/functional verification.

8. **Epic scope filtering** - Only epics and their children are processed. Non-front-end issues are recorded as out of scope and never re-evaluated.

9. **Research knowledge accumulation** - Every research phase writes a per-issue file and updates a compiled reference. Future research agents receive prior findings to avoid redundant discovery.
