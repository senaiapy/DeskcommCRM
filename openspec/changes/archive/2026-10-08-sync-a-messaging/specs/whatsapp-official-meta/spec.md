## ADDED Requirements

### Requirement: Template preview substitutes each slot at its own address
`renderTemplatePreview()` in `lib/channels/meta/render-template.ts` SHALL return the header, media header (format and link), body, footer, buttons and carousel cards of a `meta_templates` row with each `{{n}}` replaced by the value keyed by `slotKey(address, n)` of its own component and left as `{{n}}` while empty, and `renderTemplateBody()` (the text the agent's content guardrails read) SHALL be the header and body of that same preview.

#### Scenario: Same placeholder in header, body and button
- **WHEN** a template has `{{1}}` in the header, the body and a URL button, with values "Ana", "pedido 42" and "abc" for those three slots
- **THEN** the preview shows "Ana" in the header, "pedido 42" in the body and the button URL ending in "abc"

#### Scenario: Unfilled slot
- **WHEN** the body slot `{{2}}` has no value
- **THEN** the preview body still contains `{{2}}`
