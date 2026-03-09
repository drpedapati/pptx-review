# pptx-review Read Mode JSON Schema

Output of `pptx-review input.pptx --read --json`.

## Top-level

```json
{
  "slides": [ SlideInfo... ],
  "slide_count": 4
}
```

## SlideInfo

```json
{
  "number": 1,
  "layout": "Title Slide",
  "shapes": [ ShapeInfo... ],
  "notes": "Speaker notes text or empty string",
  "comments": [ CommentInfo... ]
}
```

- `number`: 1-indexed slide number
- `layout`: Slide layout name (e.g., "Title Slide", "Title and Content", "Blank")
- `notes`: Speaker notes text; empty string if none
- `comments`: Array of comments on this slide

## ShapeInfo

```json
{
  "name": "Title 1",
  "type": "textbox",
  "text": "Shape text content"
}
```

- `name`: Shape name from PowerPoint (e.g., "Title 1", "Content Placeholder 2")
- `type`: Shape type — `textbox`, `picture`, `group`, `table`, `chart`, `other`
- `text`: Concatenated text content; empty string for non-text shapes

## CommentInfo

```json
{
  "author": "Dr. Smith",
  "text": "Comment text",
  "date": "2026-03-08T12:00:00Z"
}
```

## Diff Mode JSON Schema

Output of `pptx-review --diff old.pptx new.pptx --json`.

### Top-level

```json
{
  "old_file": "old.pptx",
  "new_file": "new.pptx",
  "metadata": { "changes": [ MetadataChange... ] },
  "slides": {
    "added": [ SlideInfo... ],
    "deleted": [ SlideInfo... ],
    "modified": [ SlideModification... ]
  },
  "summary": { SummaryStats }
}
```

### SlideModification

```json
{
  "old_number": 1,
  "new_number": 1,
  "layout": "Title Slide",
  "shapes_added": [ ShapeInfo... ],
  "shapes_deleted": [ ShapeInfo... ],
  "shapes_modified": [ ShapeModification... ],
  "notes_change": { "old": "...", "new": "..." },
  "comments_added": [ CommentInfo... ],
  "comments_deleted": [ CommentInfo... ],
  "images_added": [ ImageInfo... ],
  "images_deleted": [ ImageInfo... ]
}
```

### ShapeModification

```json
{
  "name": "Title 1",
  "type": "textbox",
  "old_text": "Old Title",
  "new_text": "New Title",
  "word_changes": [
    { "type": "replace", "old": "Old", "new": "New", "position": 0 }
  ]
}
```

### SummaryStats

```json
{
  "slides_added": 0,
  "slides_deleted": 0,
  "slides_modified": 1,
  "shapes_added": 0,
  "shapes_deleted": 0,
  "shapes_modified": 1,
  "notes_changed": 0,
  "comment_changes": 0,
  "image_changes": 0,
  "metadata_changes": 0,
  "identical": false
}
```
