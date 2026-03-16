# Subissue 8: Deferred Accessibility Fixes

Remaining accessibility issues from the audit in subissue-3 that require deeper architectural work.

## 2.3 — RangeBuilder keyboard navigation is incomplete (Major)

**Problem:** RangeBuilder has three interaction zones — a "Fixed value" input+button, a list of shortcut `role="option"` items, and a "Custom range" pair of inputs+button — all wrapped in a single `role="listbox"` container. This is semantically incorrect: `listbox` children must be `option` elements, not form controls.

Keyboard navigation only works for the shortcut options (ArrowUp/Down, handled by the autocomplete plugin at `autocomplete.svelte.ts:621-646`). Tab is intercepted by the plugin to accept the selected suggestion (`autocomplete.svelte.ts:649-661`), so users cannot Tab into the input fields. The only way to reach the fixed-value or custom-range inputs is with a mouse click.

**Files:** `RangeBuilder.svelte`, `autocomplete.svelte.ts`

**Recommended approach:**
1. Change RangeBuilder's role from `listbox` to `dialog` (or remove the role and use a composite pattern with labeled sections).
2. Add a range-stage-specific code path in `handleKeyDown` that does **not** intercept Tab, allowing natural Tab order through: fixed input → Insert button → shortcut options → start input → end input → Insert button.
3. ArrowUp/Down should only navigate the shortcut options when one of them has focus, not when an input has focus.
4. Escape should still dismiss the entire range builder.

## 3.2 — Chips are not independently keyboard-focusable (Major)

**Problem:** The chip wrapper `<span>` created in `nodeviews.svelte.ts:46-51` has `contentEditable="false"` but no `tabindex` and no ARIA role. ProseMirror's NodeSelection visually highlights the chip and triggers `selectNode()`, but this is internal to ProseMirror — it doesn't move DOM focus to the chip element. Screen readers in browse/virtual-cursor mode skip the chip entirely.

**Files:** `nodeviews.svelte.ts`, `autocomplete.svelte.ts` (for ARIA attribute coordination)

**Why this is hard:** Adding `tabindex="-1"` and calling `this.dom.focus()` in `selectNode()` would break ProseMirror's focus model — the editor `<div>` must retain DOM focus for typing to work. ProseMirror expects to own focus at all times; moving it to a child node would cause the editor to fire blur events.

**Recommended approach:** Use the `aria-activedescendant` pattern (same pattern already used for autocomplete suggestions):
1. Give each chip wrapper an `id` (e.g., `search-chip-{pos}`).
2. Add `tabindex="-1"` to the wrapper so it's a valid `aria-activedescendant` target.
3. When `selectNode()` fires, set `aria-activedescendant` on the editor DOM to point at the chip's `id`. The chip's `aria-label` (already added in subissue-3) will then be announced.
4. When `deselectNode()` fires, remove `aria-activedescendant` from the editor.
5. IDs need to update when the document changes (chip positions shift), so they may need to be based on a stable identifier rather than PM position.

**Note:** This approach solves 3.3 as well — `aria-activedescendant` pointing at a labeled element triggers an automatic screen reader announcement.

## 3.3 — Chip selection state is not announced (Major)

**Problem:** When a user arrows into a chip, `selectNode()` (`nodeviews.svelte.ts:69-77`) adds a CSS class and opens ChipEditor, but nothing announces to screen readers that a chip was selected, what it contains, or what actions are available.

**Files:** `nodeviews.svelte.ts`

**Recommended approach (standalone, if 3.2 is not done):**
1. Create a shared `aria-live="polite"` region (could reuse the one already created by the autocomplete plugin, or create a new one at the SearchEditor level).
2. In `selectNode()`, push an announcement: e.g., "Selected: user: Mitchell Kotler. Press Backspace to delete or ArrowDown to edit."
3. In `deselectNode()`, clear the announcement.

**Recommended approach (with 3.2):** If `aria-activedescendant` is implemented per 3.2, this issue is solved automatically — the screen reader announces the pointed-to element's `aria-label` when `aria-activedescendant` changes.

## 5.2 — Decoration contrast needs verification (Minor)

**Problem:** CSS custom properties used for decoration colors need to be checked against WCAG 2.1 AA contrast requirements (4.5:1 for normal text, 3:1 for large text). The actual hex values depend on the theme.

**Color pairings to verify:**

| Element | Background | Text color | Notes |
|---------|-----------|------------|-------|
| AND/OR/NOT operators | `--purple-1` | inherited (likely `--gray-5`) | Background decoration on inline text |
| Parentheses | `--gray-1` | inherited | Background decoration on inline text |
| Required prefix `+` | — | `--green-3` | Text color against white editor background |
| Excluded prefix `-` | — | `--orange-3` | Text color against white editor background |
| FieldValue chip | `--blue-1` bg, `--blue-2` border | `--blue-5` text | Chip background + text |
| Range chip | `--blue-1` bg, `--blue-2` border | `--blue-5` text | Same as FieldValue |
| Sort chip | `--purple-1` bg, `--purple-2` border | `--purple-5` text | Purple variant |
| Chip field label | chip bg | chip text at 0.7 opacity | Reduced opacity lowers effective contrast |

**Files:** `SearchEditor.svelte` (decoration styles), `FieldValueChip.svelte`, `RangeChip.svelte`, `SortChip.svelte`, and the theme/variable definitions.

**Recommended approach:**
1. Resolve each CSS variable to its hex value in the default (light) theme.
2. Calculate contrast ratios using WCAG formula or a tool like [WebAIM Contrast Checker](https://webaim.org/resources/contrastchecker/).
3. For elements with reduced opacity (`.chip-field` at `opacity: 0.7`), calculate the effective color by alpha-blending against the background.
4. Document results and fix any pairings that fall below 4.5:1 (or 3:1 for large/bold text).
