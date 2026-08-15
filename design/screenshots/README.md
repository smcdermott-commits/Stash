# Design

UI/UX design for Stash, prototyped in Figma before implementation.

## Figma

Live file: [[link to Figma project](https://www.figma.com/make/7JU7cSAccre22DSuqDcYBX/Stash?p=f&fullscreen=1)]

## Status

These are Figma mockups, not screenshots of the built app. Implementation is in progress — see the root README for current build status.

## Screens

| File | Screen | Notes |
|---|---|---|
| `library.png` | Home / Library | Default view, grid layout |
| `post-detail.png` | Post detail | Shows AI summary, tags, resources |
| `share-flow.gif` | Capture flow | Instagram → Share → "Stashed!" overlay |
| `diagnostics.png` | Diagnostics | System health / processing status |

## Design Principles

- Capture should be near-instant — no blocking UI while a post saves
- Processing status should always be visible, never silent
- Collections are single-assignment; resources support multiple libraries
