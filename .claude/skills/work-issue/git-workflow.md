# Git workflow conventions

## Branch naming

Use the branch name suggested by Linear. This is available in the issue details via the `get_issue` MCP tool (look for the `branchName` or git branch field).

## Base branch logic

| Issue type | Base branch | PR target |
|------------|-------------|-----------|
| Parent (no parent issue) | `main` | `main` |
| Child (has parent issue) | Parent's branch | Parent's branch |

## Parent branch prerequisite

Before creating a child branch, always verify the parent branch exists:

```
git ls-remote --heads origin <parent-branch-name>
```

If it does not exist:
1. Fetch the parent issue from Linear
2. Get its Linear-suggested branch name
3. Create the branch from `main`: `git checkout -b <parent-branch> main`
4. Push to remote: `git push -u origin <parent-branch>`
5. Create a draft PR for the parent targeting `main`
6. Then create the child branch from the parent branch

## PR conventions

- Title matches the Linear issue title
- Body includes a reference to the Linear issue (e.g. "Resolves ISS-101")
- Created as draft initially
- Marked as ready for review only after validation passes

## Commit messages

Format: `feat(ISS-XXX): short description`

Use conventional commit prefixes:
- `feat` for new sections, blocks, or configurations
- `fix` for corrections
- `chore` for non-functional changes (locales, config cleanup)

Always include the issue ID in parentheses.

## After work is complete

- PR description is updated with a summary of all changes
- PR is marked as ready for review (remove draft status)
- Do not merge - human review happens first
