# Project conventions

## What this project is

A Shopify Horizon theme configured through JSON files, with work tracked in Linear. The full workflow design is in [WORKFLOW.md](WORKFLOW.md).

## Scope

This milestone is **configuration only**. Work consists of finding existing Horizon sections, blocks, and templates and configuring them through JSON. Do not edit Liquid, CSS, or JavaScript files.

## Skills

Three skills drive the workflow. Invoke them with `/work-issue`, `/work-epic`, or `/work-board`. See `.claude/skills/` for the full prompts.

## File conventions

- **Only edit JSON files** — templates (`templates/*.json`), sections config (`sections/*.json`), config (`config/settings_data.json`), and locales (`locales/*.json`)
- **Do not create new section or block Liquid files** — use what Horizon provides
- **Ticket map** lives at `.claude/ticket-map.md` (not committed to git)
- **Research artifacts** go in `docs/horizon-research/` (not committed to git)

## Git conventions

- Branch names come from Linear's suggested branch name
- Commit format: `feat(ISS-XXX): short description` (see `.claude/skills/work-issue/git-workflow.md`)
- PRs are created as drafts and marked ready only after validation passes
- Child issue PRs target the parent branch; parent issue PRs target `main`

## Validation

Before marking any issue as complete:

1. Run `npm run check` (Shopify theme linter)
2. Use Shopify MCP `validate_theme` on modified files
3. Check accessibility against `.claude/rules/*-accessibility.md` standards (if present)
4. Verify all acceptance criteria from the issue are met

See `.claude/skills/work-issue/validation-checklist.md` for the full checklist.

## MCP servers

- **Shopify Dev MCP** — theme validation, docs search, API reference
- **Linear MCP** — issue fetching, status updates, comments
