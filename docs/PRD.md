# Stash — Product Requirements Document

## Summary

Stash is a personal, cloud-synced archive for saving and rediscovering content shared from Instagram, starting with Reels and carousel posts. Content is captured with minimal friction, processed in the background, and made fully searchable. The product is designed to work well as a single-user personal tool while preserving the option to scale into a multi-user product later.

## Problem

Existing tools for saving Instagram content have several limitations:

- Weekly upload limits
- Local-only storage (no cloud backup, no cross-device access)
- Carousel slides saved out of order or duplicated
- No way to search saved content by summary, tag, or topic
- Websites/resources mentioned in posts are buried inside individual posts instead of being independently browsable

The underlying problem: people save large amounts of content but rarely retrieve it later. Native "saved" folders on platforms like Instagram are unsearchable and become useless past a few dozen items.

## Goals

- Capture content from Instagram with near-zero friction (share and immediately return to scrolling)
- Guarantee that a save is never lost, even if downstream processing (media download, AI enrichment) fails
- Make saved content genuinely searchable — by summary, tag, caption, or extracted resource
- Extract websites/resources mentioned in posts into their own independent, searchable collection
- Keep infrastructure at effectively $0 for personal use
- Build the architecture so it does not have to be rebuilt if the project later expands beyond personal use

## Non-Goals (V1)

- Multi-user accounts, billing, or public distribution
- Guaranteed long-term stability of Instagram scraping/extraction
- Supporting ingestion sources other than Instagram
- Android support
- Enterprise-grade security, compliance, or infrastructure

## Users

V1: a single user (the builder). Designed so that user-ownership concepts (`user_id` on records) exist in the data model from the start, without building account/auth infrastructure prematurely.

## Core User Flow

1. User is scrolling Instagram and finds a Reel or post worth saving.
2. User taps Share → Stash.
3. A lightweight overlay appears on top of Instagram (not the full app) showing "Stashed!" — the post is saved immediately, using a default collection.
4. User can optionally change the collection from the overlay, or dismiss it (tap Done or swipe down) and keep scrolling.
5. In the background, Stash downloads the media, generates an AI title/summary/tags, and extracts any mentioned resources/links.
6. Later, the user opens the Stash app to search, browse by collection, or review saved resources.

## Feature Requirements

### 1. Capture (Share Extension)

- Native iOS Share Extension triggered from Instagram's system share sheet
- Post is saved immediately on share, independent of processing success
- Default collection is applied automatically; user can change it inline without leaving the overlay
- No "Save" button required — selecting a different collection updates it immediately
- Dismissible at any time (Done button or swipe down)

### 2. Background Processing

- Every saved post has a processing status, tracked at the level of individual components (media, thumbnail, AI summary, tags, resources), not just an overall pass/fail
- Media download and AI enrichment run asynchronously and do not block the save
- Automatic retry with exponential backoff for transient failures
- Manual "Complete Upload" action for posts that need attention, which retries only the missing/failed components
- Carousel media stored as an explicit, position-indexed, uniquely-ID'd list to prevent reordering or duplication

### 3. AI Enrichment

- Auto-generated title
- Auto-generated summary (configurable length)
- Auto-generated tags
- Provider-agnostic AI integration (not hard-coded to one vendor/model)

### 4. Resource Extraction

- URLs mentioned in a post's caption/content are detected automatically
- Each resource is stored as its own record: URL, name, category, description, source post(s), first-discovered date
- Resources are browsable and searchable independently of the posts that mentioned them
- A single resource can be linked to multiple posts

### 5. Organization

- **Collections** (folders for posts) — each post belongs to at most one collection. `collection_id` is nullable; posts with no collection are "unsorted" rather than forced into a folder.
- Default save behavior is configurable: leave new posts unsorted, or auto-assign them to a user-chosen default collection. Either way, the collection can be changed at any time, including from the capture overlay.
- The Library/Home view must include an explicit "Unsorted" or "All" filter so unassigned posts remain visible and discoverable, not just accessible by accident.
- **Libraries** (folders for resources) — a resource can belong to multiple libraries (many-to-many), since resources are more naturally cross-cutting than posts (e.g., a single website can reasonably belong to both "Learning" and "Programming").
- Tags — many-to-many, apply to posts, can be AI-generated or manual
- Optional personal note per post ("why did I save this?")

### 6. Search

- Full-text search across titles, summaries, captions, tags, personal notes, and resource descriptions
- Search should return relevant posts and resources together, not just posts

### 7. Diagnostics

- System health view showing ingestion success rate over time (e.g., last 24 hours vs. last 7 days)
- Structured error records per processing job: operation, status, error type, attempt count, timestamps
- Detection of abnormal failure spikes (e.g., success rate drops significantly), distinct from isolated single-post failures
- Push notifications for processing issues, intelligently grouped (e.g., "7 posts failed in the last 10 minutes" rather than one notification per failure)
- Recovery notifications when processing returns to normal

### 8. Settings

- Default collection and default share behavior
- AI processing toggles (titles, summaries, tags, resource detection)
- Notification preferences, grouped by category
- Search preferences (which fields to include, default sort)
- Storage usage view
- Data export
- Destructive actions (delete all data) isolated in a clearly marked section

### 9. Cross-Device Access

- Primary interface: native mobile app (iOS first)
- Secondary interface: web access to the same data, for use on a computer

## Data Model (Conceptual)

```
User
Collection
Post (source, source_url, title, summary, caption, collection_id [nullable], status)
Media (post_id, type, position, storage_key)
Tag / PostTag
Resource
Library
ResourceLibrary (resource_id, library_id)   — many-to-many
PostResource (post_id, resource_id)         — many-to-many
ProcessingJob (post_id, operation, status, attempts, error, timestamps)
```

Posts have a single, optional (nullable) collection — a post is either filed under one folder or left unsorted. Resources may belong to multiple libraries, and are independently linked to every post that mentions them via `PostResource`. These are two separate many-to-many relationships on `Resource`, not one.

All user-owned entities include a `user_id` field from the start, even with a single user, to avoid retrofitting ownership later.

## Architecture Principles

- **Ingestion is decoupled from the core app.** Instagram is treated as one replaceable ingestion adapter, not a foundation. The data model and UI should make sense even if a source is temporarily broken.
- **Saving and processing are separate operations.** A save is durable the moment the URL and metadata are recorded; processing failures never lose the underlying post.
- **Secrets never live in the frontend.** All API keys and credentials are server-side only.
- **The backend never trusts the frontend.** Every operation is checked against resource ownership server-side.
- **Submitted URLs are treated as untrusted input** (SSRF prevention: no requests to localhost, private IP ranges, or cloud metadata endpoints).
- **Media storage is private by default**, accessed only via signed/temporary URLs.

## Technical Stack

- Mobile app: React Native + Expo (TypeScript)
- Backend: Cloudflare Workers
- Database: Cloudflare D1
- Media storage: Cloudflare R2
- AI: provider-agnostic interface
- Source control / CI: GitHub, GitHub Actions

## Risks

| Risk | Severity | Mitigation |
|---|---|---|
| Instagram extraction breaks due to platform changes | High | Isolated, replaceable ingestion adapter; save/process decoupling; diagnostics to detect failures quickly |
| Media/AI processing costs grow with usage | Medium | Monitor storage growth; provider-agnostic AI to allow cheaper models |
| Cloud provider outage or account loss | Medium | Periodic data export/backup as a first-class feature |
| Scope creep before validating the core workflow | Medium | Ship a minimal V1 (capture → save → search) before adding AI/resources/diagnostics |
| SSRF via submitted URLs | Medium | Input validation, blocked internal address ranges |

## Success Criteria (V1)

- Can share a Reel/carousel from Instagram and have it saved without leaving Instagram
- Saved posts are visible and searchable in the Stash app within a reasonable time after processing
- A failed media download does not lose the post record, and can be retried
- Carousels are stored in correct order with no duplicates
- Personal usage over an extended period (weeks) without data loss

## Future Considerations (Not V1)

- Android support
- Additional ingestion sources (YouTube, general web links, manual uploads)
- Multi-user accounts and authentication
- Payments/subscription tiers
- Public release
