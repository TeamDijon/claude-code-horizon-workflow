# Claude Code Skills for Shopify Horizon Theme Configuration

A reference implementation of [Claude Code skills](https://docs.anthropic.com/en/docs/claude-code/skills) that turn Linear issues into Shopify Horizon theme changes, end to end.

Three skills handle increasing levels of autonomy:

| Skill | What it does |
|-------|-------------|
| `/work-issue ISS-101` | Process a single issue through five phases: setup, analysis, research, implementation, validation |
| `/work-epic ISS-100` | Batch-process all children of a parent issue (or a targeted set of issues) |
| `/work-board` | Scan the full Linear board, categorize issues, propose a work order, and execute |

## How it works

```
/work-board
│
├── Queries Linear MCP → builds a ticket map (local markdown cache)
├── Scans open PRs via gh CLI for review feedback
├── Categorizes issues (ready / re-work / blocked / done / out of scope)
├── Proposes a work order → user approves
│
└── For each issue:
    ├── Phase 1 - Setup: create branch + draft PR, link to Linear
    ├── Phase 2 - Analysis: read issue, summarize requirements
    ├── Phase 3 - Research: Explore subagent searches codebase + Shopify docs
    ├── Phase 4 - Implementation: general-purpose subagent edits JSON config
    └── Phase 5 - Validation: theme linter + Shopify validation + accessibility audit
```

The agent delegates heavy work to subagents (research, implementation, validation) and keeps the main conversation as a lean orchestrator. A ticket map markdown file provides session continuity so work can resume across conversations.

## Repo structure

```
.
├── CLAUDE.md                          # Project conventions for Claude Code
├── WORKFLOW.md                        # Full workflow design document
├── .mcp.json                          # MCP server config (Shopify + Linear)
├── .mcp.template.json                 # Shareable MCP template (no secrets)
│
├── .claude/
│   └── skills/
│       ├── work-issue/
│       │   ├── SKILL.md               # Single-issue workflow (5 phases)
│       │   ├── git-workflow.md        # Branch and PR conventions
│       │   └── validation-checklist.md # Linting, validation, accessibility checks
│       ├── work-epic/
│       │   └── SKILL.md               # Batch orchestrator
│       └── work-board/
│           └── SKILL.md               # Board scanner and planner
│
└── docs/
    ├── ticket-map.md                  # Example ticket map
    └── horizon-research/
        └── horizon-components.md      # Example compiled research reference
```

## Prerequisites

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) CLI or VS Code extension
- [Shopify Dev MCP](https://github.com/Shopify/dev-mcp) for theme validation and docs
- [Linear MCP](https://mcp.linear.app) for issue tracking
- [GitHub CLI](https://cli.github.com/) (`gh`) for PR operations
- A Shopify store with the [Horizon theme](https://themes.shopify.com/themes/horizon) installed

## Setup

1. **Clone the repo** into your Shopify theme project directory, or copy the `.claude/skills/`, `CLAUDE.md`, and `WORKFLOW.md` files into an existing project.

2. **Configure MCP servers.** Copy `.mcp.template.json` to `.mcp.json` and authenticate:
   - Shopify Dev MCP: runs via npx, no extra config needed
   - Linear MCP: connect via OAuth on first use (`npx mcp-remote https://mcp.linear.app/mcp`)

3. **Customize for your project:**
   - Update `CLAUDE.md` with your project name, team conventions, and scope rules
   - Update the default milestone name in `.claude/skills/work-board/SKILL.md` if desired
   - Add accessibility rules to `.claude/rules/` if your project has specific standards

4. **Start working:**
   ```
   /work-board                    # Scan the board and plan
   /work-epic ISS-100             # Process all children of a parent issue
   /work-issue ISS-101            # Work on a single issue
   ```

## Adapting to your project

The skills are designed around a specific pattern — Linear issues driving Shopify Horizon JSON configuration — but the underlying structure is reusable:

- **Different issue tracker:** Replace Linear MCP calls with your tracker's MCP or API. The ticket map format stays the same.
- **Different theme/framework:** Replace Shopify MCP research with your framework's docs. Swap `npm run check` for your linter.
- **Broader scope:** The current milestone is configuration-only (JSON edits). To add Liquid/CSS/JS work, remove the JSON-only constraint from the implementation phases and update the validation checklist.
- **Different branching model:** Edit `git-workflow.md` to match your conventions. The parent/child branch hierarchy maps well to any epic/subtask structure.

## Key design decisions

See [WORKFLOW.md](WORKFLOW.md) for the full rationale. Highlights:

1. **Configuration only** — This milestone configures existing Horizon components through JSON. No custom code.
2. **Research gates implementation** — If research can't find a clear path, the agent stops and asks rather than guessing.
3. **Flat subagent hierarchy** — Main conversation orchestrates. Subagents do focused work and return summaries.
4. **Ticket map as local cache** — Avoids repeated MCP queries. Enables session continuity.
5. **Human handoff at "In Review"** — The agent's job ends when the PR is ready. Humans handle visual/functional verification.

## License

MIT
