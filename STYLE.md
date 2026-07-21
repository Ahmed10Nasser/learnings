# Style guide

Conventions for writing and organizing notes in this repo. Keep it lightweight — the
goal is consistency, not bureaucracy.

## File naming

- Use `kebab-case.md` for every note (e.g. `covering-indexes.md`, `observer-pattern.md`).
- Names should be descriptive and specific — the file name is the note's title in the index.
- One concept per file. If a note grows to cover several ideas, split it.

## Note shape

There's no rigid template, but a good note generally has:

1. A single `# H1` title matching the topic.
2. A one-line summary right under the title.
3. The body — explanation, examples, diagrams, code snippets as needed.
4. A `## References` section at the end linking sources (articles, books, docs).

## Cross-linking

Link related notes with relative Markdown links, e.g.
`See [covering indexes](../databases/covering-indexes.md).` This keeps the repo
navigable as a connected web of notes.

## Where a note goes

- Add the note to the existing topic folder that best fits it.
- Create a **new top-level folder** only when a distinct domain emerges that doesn't
  fit any current one — not for a single note. Prefer adding to `best-practices/` or
  the closest existing topic first.

## Keeping indexes current

Each topic folder's `README.md` is a manually maintained index. **When you add a
note, add a bullet linking to it** in that folder's `README.md` (and, if it's a new
folder, link the folder from the root [README](README.md)).
