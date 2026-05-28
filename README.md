# language-reader-blueprint

A single-file recipe for building a personal, taste-matched **reading practice
system** for any language you're learning. Hand `BLUEPRINT.md` to an AI agent,
answer the interview, and walk away with a self-hosted reader of short stories
in your target language — annotated with hover-popup vocabulary in your native
language.

## What you get when you implement it

- A **stable project index** (`index.html`) — the table of contents for your reader.
- **Per-story folders** at `stories/NN-slug/` containing:
  - `story.md` — pure-target-language story, 150–500 words, calibrated to your level.
  - `enrichment.toml` — per-word translations, parts of speech, lemmas, and
    optional grammar notes; per-sentence translations and optional sentence
    notes; idiom entries with a whole-meaning gloss plus a word-by-word breakdown.
  - `index.html` — bespoke single-file HTML with hover popups (word, idiom, and
    full-sentence — the last also via Shift-hover on any word), hand-designed
    per story (no shared template).
- **Three reusable skills** at `.ai/skills/{story,enrich,render}/` that
  encapsulate the pipeline: write → annotate → render.

Everything is plain files on disk. No build step, no frameworks, no servers.
Open the HTML files in a browser.

## How it works

1. The agent reads `BLUEPRINT.md`.
2. It interviews you in your native language about: level, taste, anti-vibes,
   reference works that resonate, shared-universe preference, format choices.
3. It scaffolds the project and writes your `profile.md`.
4. You ask for stories on demand. Each story flows through the three skills.

The blueprint is opinionated only where it costs nothing to be (see the
"Critical invariants" section). Everything else — palette, typography, story
length, vocabulary ratio, grammar depth, dictionary URL, whether to use
external fonts, whether to show sentence translations — is dial-able via the
profile.

## Usage

Open an empty folder in your AI agent of choice and paste:

> Fetch https://github.com/usukololgubu/language-reader-blueprint/blob/main/BLUEPRINT.md
> and follow its instructions to set up a language-reader project in this folder.

## A live instance

A working Russian-native / Spanish-A1 implementation built from this blueprint
lives at: <https://github.com/usukololgubu/lee-espanol>
