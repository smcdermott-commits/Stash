# Stash

Personal, cloud-synced archive for saving and searching content shared from Instagram (and eventually other sources).

## Problem

Saved content tends to disappear into a folder no one revisits. Stash's goal is retrieval, not just saving: every post is processed into a searchable record (title, summary, tags, extracted links) instead of sitting in an unsearchable pile.

## Why this exists

Built to replace an existing Reels-saving apps that have weekly upload limit, no cloud backup, inconsistent carousel ordering, and no search. Also wanted links mentioned in posts pulled into their own searchable collection instead of being buried in individual saves.

## Features

- Share from Instagram, saved instantly; processing happens in the background
- Saving is decoupled from media processing; a failed download doesn't lose the post
- Automatic retry with backoff, plus manual retry for anything that needs it
- AI-generated title, summary, and tags per post
- Extracted links stored as a separate, searchable resource library
- Full-text search across titles, summaries, captions, tags, and notes
- Position-indexed media storage to prevent carousel reordering/duplication
- Diagnostics view tracking ingestion success rate over time
- Mobile app + (future) web access to the same data

## Architecture

```
Instagram
   │ Share
   ▼
iOS Share Extension
   │
   ▼
Stash API ──► save record immediately (URL, collection, timestamp)
   │
   ▼
Background processing
   ├── media download + ordering
   ├── AI summary / title / tags
   └── link extraction
   │
   ▼
Database + media storage
   │
   ▼
Search (mobile + web)
```

Saving a post and processing its media are separate operations. If Instagram changes something and extraction breaks, only the processing step needs retrying — the saved record isn't affected.

## Stack

- Mobile: React Native + Expo (TypeScript)
- Backend: Cloudflare Workers
- Database: Cloudflare D1
- Media storage: Cloudflare R2
- AI: provider-agnostic interface
- CI: GitHub Actions

## Docs

| File | Contents |
|---|---|
| [`docs/PRD.md`](./docs/PRD.md) | Requirements and scope |
| [`docs/ARCHITECTURE.md`](./docs/ARCHITECTURE.md) | System design, data flow |
| [`docs/DESIGN.md`](./docs/DESIGN.md) | UX flows, Figma reference |
| [`docs/DATABASE.md`](./docs/DATABASE.md) | Schema |
| [`docs/DECISIONS.md`](./docs/DECISIONS.md) | Architecture decision records |
| [`docs/DEVELOPMENT.md`](./docs/DEVELOPMENT.md) | Build log |
| [`docs/CHANGELOG.md`](./docs/CHANGELOG.md) | Version history |

## Notable engineering problems

**Unreliable upstream source.** Instagram has no stable public API for arbitrary post extraction. The ingestion layer is isolated so it can be replaced or patched without touching the rest of the app.

**Async, resumable processing.** Media download and AI processing run independently of the save action, with exponential-backoff retry and a manual "Complete Upload" fallback.

**Carousel integrity.** Media is stored as a position-indexed list with unique IDs per item rather than trusting source order/count — this is what caused ordering/duplication bugs in the app Stash replaces.

**Failure observability.** A diagnostics layer tracks per-job outcomes and success rate over time, so a spike in failures shows up as a trend rather than being discovered post-by-post.

**Cost constraints.** Built to run on free-tier infrastructure; media storage identified as the likeliest long-term cost, not the database.

## Status

Early development.

**Done:** requirements, architecture design, UI design (Figma)
**In progress:** data model, backend API, Instagram share capture, background processing
**Planned:** AI enrichment, search, diagnostics dashboard, notifications
**Later:** additional ingestion sources, Android, data export

## License

TBD
