---
name: pptx-review
description: "Read, edit, and diff PowerPoint presentations (.pptx) with text replacement, speaker notes, slide manipulation, comments, and semantic diffing using the pptx-review CLI v1.2.0 — a .NET 8 tool built on Microsoft's Open XML SDK. Ships as a single 14MB native binary (no runtime). Use when: (1) Replacing or setting text in existing slides, (2) Managing speaker notes on any slide, (3) Adding, deleting, duplicating, or reordering slides, (4) Adding comments to slides, (5) Reading/extracting slides, shapes, notes, comments, and metadata from a .pptx, (6) Diffing two .pptx files semantically, (7) Revising presentation content for review, (8) Any task requiring programmatic PowerPoint editing with formatting preservation that renders natively in PowerPoint."
---

# pptx-review v1.2.0

CLI tool for PowerPoint presentation editing: text replacement, speaker notes, slide manipulation, comments, read, diff, and git integration. Built on Microsoft's Open XML SDK — 100% compatible with PowerPoint.

## Install

```bash
brew install drpedapati/tools/pptx-review
```

Binary: `/opt/homebrew/bin/pptx-review` (14MB, self-contained, no runtime)

Verify: `pptx-review --version`

## Workflow Decision Tree

- **Reading/extracting content?** → `pptx-review input.pptx --read --json`
- **Editing text, notes, or slides?** → Build JSON manifest → `scripts/review_pipeline.sh`
- **Comparing two versions?** → `pptx-review --diff old.pptx new.pptx`
- **Git-friendly diffs?** → `pptx-review --textconv presentation.pptx`

## Modes

### Edit: Apply changes and comments

Takes a `.pptx` + JSON manifest, produces an edited `.pptx`.

```bash
pptx-review input.pptx edits.json -o edited.pptx
pptx-review input.pptx edits.json -o edited.pptx --json    # structured output
pptx-review input.pptx edits.json --dry-run --json          # validate without modifying
cat edits.json | pptx-review input.pptx -o edited.pptx      # stdin pipe
pptx-review input.pptx edits.json -o edited.pptx --author "Dr. Smith"
pptx-review input.pptx edits.json -i                                # edit in-place (rollback-safe)
```

### Read: Extract presentation content as JSON

```bash
pptx-review input.pptx --read --json
```

Returns: slides (with layouts), shapes (name/type/text), speaker notes, comments, and slide count. For output schema, see `references/read-schema.md`.

### Diff: Semantic comparison of two presentations

```bash
pptx-review --diff old.pptx new.pptx
pptx-review --diff old.pptx new.pptx --json
```

Detects: slide additions/deletions, shape text changes (word-level), speaker notes changes, comment modifications, image changes (SHA-256), metadata changes.

### Git: Textconv driver for meaningful PowerPoint diffs

```bash
pptx-review --textconv presentation.pptx    # normalized text output
pptx-review --git-setup                      # print .gitattributes/.gitconfig instructions
```

## JSON Manifest Format

Build this JSON, pass it to `pptx-review`.

```json
{
  "author": "Reviewer Name",
  "changes": [
    { "type": "replace_text", "find": "old text", "replace": "new text" },
    { "type": "replace_text", "find": "old text", "replace": "new text", "slide": 2 },
    { "type": "set_text", "slide": 1, "shape": "Title 1", "text": "New Title" },
    { "type": "set_notes", "slide": 1, "text": "Speaker notes content" },
    { "type": "delete_slide", "slide": 3 },
    { "type": "duplicate_slide", "slide": 1, "position": 5 },
    { "type": "reorder_slide", "slide": 2, "position": 4 },
    { "type": "add_slide", "layout": "Blank", "position": 2 }
  ],
  "comments": [
    { "slide": 1, "text": "Comment on this slide" }
  ]
}
```

### Change types

| Type | Required Fields | Optional | Result |
|------|----------------|----------|--------|
| `replace_text` | `find`, `replace` | `slide` | Replace text globally or on specific slide |
| `set_text` | `slide`, `shape`, `text` | — | Set exact text of a named shape |
| `set_notes` | `slide`, `text` | — | Set or replace speaker notes |
| `delete_slide` | `slide` | — | Remove slide (1-indexed) |
| `duplicate_slide` | `slide` | `position` | Clone slide with layout preserved |
| `reorder_slide` | `slide`, `position` | — | Move slide to new position |
| `add_slide` | — | `layout`, `position` | Add new slide |

### Critical rules for text matching

1. **Must be exact copy-paste from the presentation.** The tool tries exact ordinal match first, then falls back to whitespace-flexible matching (treats any whitespace run including NBSP as equivalent).
2. **Use `--read --json`** to see exact shape names and text content before editing.
3. **Scope with `slide`** when the same text appears on multiple slides.
4. **Shape names** (for `set_text`) come from the `name` field in read output (e.g., "Title 1", "Content Placeholder 2").
5. Always validate with `--dry-run --json` before applying.

### Concrete example

Given a presentation where slide 2 contains shape "Content Placeholder 2" with text: *"Participants: 50 healthy adults (25M/25F)\nMRI Protocol: 3T Siemens scanner\nAnalysis: FSL and FreeSurfer pipelines"*

```json
{
  "author": "Dr. Jones",
  "changes": [
    {
      "type": "replace_text",
      "find": "50 healthy adults",
      "replace": "75 healthy adults",
      "slide": 2
    },
    {
      "type": "set_notes",
      "slide": 3,
      "text": "Emphasize the DMN connectivity finding and its clinical relevance"
    }
  ],
  "comments": [
    {
      "slide": 1,
      "text": "Please update the department name to match the new org structure"
    }
  ]
}
```

Note: `replace_text` scoped to slide 2 for precision. `set_notes` overwrites existing notes entirely. Comments reference slides by number.

## Helper Scripts

### `scripts/validate_manifest.sh`

Dry-run validation with human-readable pass/fail summary. Run before applying edits.

```bash
scripts/validate_manifest.sh presentation.pptx edits.json
# Output: 3/3 edits matched
```

### `scripts/review_pipeline.sh`

Full pipeline: validate → apply → report output path. Aborts on validation failure.

```bash
scripts/review_pipeline.sh presentation.pptx edits.json edited.pptx
```

## JSON Output (--json)

```json
{
  "input": "presentation.pptx",
  "output": "presentation_edited.pptx",
  "author": "Dr. Smith",
  "changes_attempted": 3,
  "changes_succeeded": 3,
  "comments_attempted": 1,
  "comments_succeeded": 1,
  "success": true,
  "results": [
    { "index": 0, "type": "comment", "success": true, "message": "Comment added to slide 1" },
    { "index": 0, "type": "replace_text", "success": true, "message": "Replaced 1 occurrence(s) on slide 2" }
  ]
}
```

Exit code 0 = all succeeded. Exit code 1 = at least one failed (partial success possible).

## Key Behaviors

- **Whitespace-flexible matching.** Exact ordinal match tried first; if that fails, falls back to a regex that normalizes whitespace runs (spaces, NBSP, tabs). Compiled regexes are cached.
- **Formatting preserved.** RunProperties cloned from source runs onto replacement text.
- **Multi-run text matching.** Text spanning multiple XML runs (including breaks within runs) is found and handled correctly.
- **In-place editing is rollback-safe.** Uses temp file + atomic move; original untouched on failure.
- **Comments applied first**, then changes. Ensures slide structure is stable during comment insertion.
- **Slide operations are order-sensitive.** Delete/reorder changes execute sequentially — plan slide numbers accordingly.
- **Everything untouched is preserved.** Images, charts, animations, transitions, embedded objects, masters, and layouts survive intact.

## Companion Tools

| Tool | Install | Purpose |
|------|---------|---------|
| `docx-review` | `brew install drpedapati/tools/docx-review` | Word document review with tracked changes |
| `xlsx-review` | `brew install drpedapati/tools/xlsx-review` | Excel read/edit |

Same architecture: .NET 8, Open XML SDK, single binary, JSON in/out.
