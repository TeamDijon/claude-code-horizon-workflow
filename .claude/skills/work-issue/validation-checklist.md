# Validation checklist

Run all checks before marking an issue as complete. All must pass.

## Theme linter

Run `npm run check` (which executes `shopify theme check`).

All errors must be resolved. Warnings should be reviewed - fix if straightforward, document if intentional.

## Shopify theme validation

Use the Shopify MCP `validate_theme` tool. Pass:
- `absoluteThemePath`: the full path to the project root
- `filesCreatedOrUpdated`: array of objects with `path` for each modified file (relative paths)

All validation errors must be resolved.

## Accessibility audit

For each modified file, check against the relevant accessibility standards in `.claude/rules/`:

| If the change involves... | Check against |
|---------------------------|---------------|
| Images | `image-alt-text.md` |
| Headings | `heading-accessibility.md` |
| Forms or inputs | `form-accessibility.md` |
| Modals or dialogs | `modal-accessibility.md` |
| Carousels or slideshows | `carousel-accessibility.md` |
| Accordions | `accordion-accessibility.md` |
| Product cards | `product-card-accessibility.md` |
| Cart drawer | `cart-drawer-accessibility.md` |
| Navigation or landmarks | `landmark-accessibility.md` |
| Animations | `animation-accessibility.md` |
| Color or contrast | `color-contrast.md` |
| Focus or keyboard | `focus-order-and-styles.md` |
| Mobile layout | `mobile-accessibility.md` |
| Page structure | `global-accessibility-standards.md` |

Since this milestone is configuration-only (JSON edits), accessibility checks focus on verifying that the chosen Horizon components and their configured settings produce accessible output.

## Requirements check

Compare the implementation against the acceptance criteria documented in Phase 2 analysis. Every criterion must be addressed.

## Validation report format

Present results as:

```
Validation report for ISS-XXX

Theme linter: PASS / FAIL (details)
Theme validation: PASS / FAIL (details)
Accessibility: PASS / FAIL (details)
Requirements: X of Y criteria met

Overall: PASS / FAIL
```
