# Personal Language Reader — Blueprint

> A copy-pasteable instruction document for AI agents. Hand this file to your agent
> and it will scaffold a personalized language-reading project for any
> (native → target) language pair, calibrated to your level and taste.
>
> Validated baseline: a working Russian-native / Spanish-A1 reader in a shared
> "Space Expansion" sci-fi universe. All architectural decisions below are
> battle-tested against that implementation.
>
> **What's negotiable.** Treat [Part 4 — Critical invariants](#part-4--critical-invariants)
> as non-negotiable; those are bugs already paid for. Everything else is
> dial-able — taste, layout, palette, fonts, tokenization granularity, grammar
> depth, even the grammar mini-language itself. The blueprint is opinionated
> where it costs nothing to be, and silent where the user's preference should
> decide. If you (or the user) want to override an invariant, surface the
> tradeoff and override it consciously — not by accident.

---

## TL;DR

You (the user) get a private folder of short stories in the language you're learning.
Each story is:

- Written **only in the target language**, calibrated to your CEFR level.
- Rendered as a **single self-contained HTML file** with a bespoke design — typography,
  color, illustration all driven by the story's mood.
- Annotated with **hover popups** on every meaningful word: translation into your
  native language, part-of-speech, dictionary lemma, and (optionally) rich grammar
  explanations.
- Tightly themed to **your** taste profile: topics you actually want to read, tones
  you actually enjoy, anti-vibes you actually want to avoid.

The AI agent that follows this blueprint will: interview you in your native language,
build the project skeleton on disk, install three reusable skills, then generate
stories you can iterate on.

---

## Table of contents

- [Part 0 — Glossary](#part-0--glossary)
- [Part 1 — Interview the user](#part-1--interview-the-user)
- [Part 2 — Project scaffold](#part-2--project-scaffold)
- [Part 3 — The three skills](#part-3--the-three-skills)
  - [3.1 story](#31-story-skill)
  - [3.2 enrich](#32-enrich-skill)
  - [3.3 render](#33-render-skill)
- [Part 4 — Critical invariants (hard-won bugs)](#part-4--critical-invariants)
- [Part 5 — Worked example (FR → EN micro-story)](#part-5--worked-example)
- [Part 6 — Anti-checklist](#part-6--anti-checklist)
- [Part 7 — Iteration loop](#part-7--iteration-loop)

---

## Part 0 — Glossary

| Term | Meaning |
|------|---------|
| **Native language** | The language popup translations and grammar explanations are written in. The language the agent uses when interviewing the user. |
| **Target language** | The language the user is learning. All story bodies are in this language only. |
| **Reader profile** | `profile.md` — the user's level, taste, anti-vibes, and shared-universe spec. The agent re-reads this before every generation. |
| **Story** | One short text. Lives in `stories/NN-slug/` together with its enrichment data and rendered HTML. |
| **Enrichment** | `enrichment.toml` — per-word translations, parts of speech, lemmas, and optional grammar; per-sentence and per-phrase translations. |
| **Render** | The hand-designed `index.html` page produced for one story. Each story gets its own design — no shared base template. The designer (LLM) writes the page freely; a small helper script then applies enrichment to wrap words and inject popup behavior. |
| **Hover popup** | The UI affordance shown when the user hovers (or taps) a wrapped word in the rendered HTML. Contains translation, POS, lemma, dictionary link, optional grammar block. |
| **Grammar mini-language** | A tiny markdown-ish syntax (`**form**`, `*term*`, `` `code` ``, `[[suffix]]`) used inside `enrichment.toml`'s `grammar` field, parsed at popup-show time by an inline JS function `mdToHtml()`. |
| **Render helper script** | Small program (Python 3.11+ in the reference implementation) that applies `enrichment.toml` to a designer-authored HTML page: walks the element marked `data-story-body`, wraps words/sentence-terminators with popup spans, auto-injects popup CSS+JS, and refreshes the project index. Re-runnable. |
| **Project index** | The root `index.html` — a reader-friendly table of contents listing all stories. |

---

## Part 1 — Interview the user

> **Important**: Conduct this interview in the user's NATIVE language, not in English.
> Detect or ask for native language first, then switch.

Ask the questions below in order. Save final answers to `profile.md`. Treat answers as
authoritative input to all subsequent generation: every story respects every line of
this file.

### Section A — Language pair

1. **What is your native language?**
   This is the language popup translations and grammar will appear in. It's also the
   language we use for this interview.

2. **What language are you learning?**
   Every story body will be in this language only.

3. **Any variant or dialect preference?**
   Examples: Latin American vs Castilian Spanish, Brazilian vs European Portuguese,
   Mainland vs Taiwanese Mandarin, Modern Standard Arabic vs Egyptian, neutral if
   unsure. Affects vocabulary and orthography.

### Section B — Level & calibration

4. **What's your current level in the target language?**
   CEFR labels (A1, A2, B1, B2, C1, C2) are convenient. Self-descriptions are fine
   too: "I know maybe 500 words", "I can read children's books slowly", "I can
   follow news articles with a dictionary".

5. **How long should each story be?**
   Defaults by level:
   - A1: 150–300 words
   - A2: 300–500 words
   - B1: 500–800 words
   - B2+: 800–1500 words

   Bias toward shorter at first — the user can ask for longer ones once pacing is
   confirmed.

6. **What proportion of new (above-level) vocabulary should each story include?**
   Default: 80% in-level, 20% above-level. Above-level words are still glossed in
   popups; the proportion controls how aggressively the user is pushed.

### Section C — Taste profile

7. **What topics genuinely interest you?**
   Push for specificity. "Sci-fi" is too broad; "hard sci-fi with realistic
   physics, no space wizards" is useful. "Cooking" is too broad; "fermentation,
   regional bread traditions, no recipe-blog filler" is useful.

8. **What do you NOT want?** (Anti-vibes.)
   Examples that come up in practice: moralizing, romance, naive children's-book
   tone, jump-scares, gratuitous violence, twee whimsy, motivational uplift,
   detective-novel reveals. Be specific — these are filters, not preferences.

9. **Reference works.**
   Ask for 2–5 books, authors, films, or shows that capture the vibe the user
   wants. Then ask **what specifically** in each one resonates — not the surface
   genre but the specific quality (e.g. "the Strugatskys' *Hard to Be a God* for
   the observer-trapped-in-alien-society dynamic, not the medievalism").

10. **Tone preference.**
    Place on a spectrum: serious ↔ ironic, contemplative ↔ kinetic, dark ↔ warm,
    realistic ↔ whimsical. Mixes are OK — record the mix ratio if relevant.

11. **Self-contained or serialized?**
    Default: self-contained (each story stands alone), but a **shared universe**
    (recurring setting, accumulating lore) is the recommended default. Pure
    one-offs with no shared spine are also fine. Cliffhangers are usually a bad
    idea for language learners — they create reading anxiety.

12. **Shared universe — what kind?**
    If yes to shared universe, briefly sketch the world: one paragraph. Save this
    to `profile.md` and reference in every story brief.

### Section D — Format & interaction

13. **Grammar in popups: rich or minimal?**
    - **Rich**: per-word grammar block with conjugation/declension breakdowns,
      formal/informal register, irregularity notes. Use the grammar mini-language
      in `enrichment.toml`. Recommended for A1–B1 learners.
    - **Minimal**: just translation + POS + lemma. Recommended for B2+ or for
      users who find rich grammar distracting.

14. **Which words should get popups?**
    - **All meaningful words** (default): every noun, verb, adjective, adverb,
      pronoun, preposition, conjunction — basically anything that carries meaning.
    - **Skip function words**: omit articles and very basic pronouns the user
      already knows cold.
    - Punctuation, numerals, and proper nouns are NEVER wrapped.

14b. **Full-sentence translations — show them?**
    The enrichment file always carries per-sentence translations. The renderer
    decides what to do with them.
    - **Off** (recommended for B2+): translations stay in the file, the page
      never surfaces them. Word-level popups only.
    - **Opt-in toggle** (recommended for A1–B1, default): a small chrome button
      reveals sentence translations inline on demand; off by default each session.
    - **Always on**: translations sit beneath each sentence permanently.

14c. **External fonts — allowed?**
    - **Yes** (default): the renderer may load up to two Google Fonts per story.
      The page degrades cleanly without network.
    - **No, system fonts only**: the renderer picks from system-available
      families (e.g. Georgia, Iowan Old Style, Cambria; or Menlo, Consolas).
      Pick this for offline-first or privacy-conscious setups.

15. **Dictionary for the "open in dictionary" link.**
    One per target language. The `{lemma}` token is the substitution point — the
    renderer replaces it with the URL-encoded lemma at popup-build time. Examples:
    - Spanish: `https://www.spanishdict.com/translate/{lemma}`
    - French: `https://www.wordreference.com/fren/{lemma}`
    - German: `https://www.dict.cc/?s={lemma}`
    - Japanese: `https://jisho.org/search/{lemma}`
    - Chinese: `https://www.mdbg.net/chinese/dictionary?wdqb={lemma}`
    - Default fallback for any language: `https://en.wiktionary.org/wiki/{lemma}`

    Ask the user which they prefer. They may have their own favorite.

### Section E — Operational

16. **Where on disk should the project live?**
    Full absolute path. Suggest the agent's current working directory as the
    default (e.g. `<cwd>/language-reader`) so the user can just hit enter. The
    agent creates the folder if absent.

17. **Cadence after scaffold?** *(ask only if the agent itself supports
    autonomous/scheduled execution — cron, background tasks, scheduled wake-ups.
    If the agent is a simple chat session with no scheduler, skip this question;
    cadence is trivially "on-demand" in that case.)*

    If asked, offer:
    - **On-demand** (default) — user asks for the next story when they want it.
    - **Scheduled drops** — agent generates a new story on a schedule (e.g.
      one per week, every weekday morning). Requires the agent to register a
      recurring task in its own scheduler. Concrete cadence is up to the user.

    Either way, the generation logic is the same — the cadence only controls
    *when* the agent kicks off the story skill, not *how*.

### Saving the interview

After the last question, write `profile.md` (template in
[Part 2](#part-2--project-scaffold)). Read it back to the user and confirm before
proceeding to scaffold the project. Then offer to generate the first story — let
the user steer the topic (or ask the agent for suggestions) rather than
pre-planning a fixed batch.

---

## Part 2 — Project scaffold

After the interview, create this layout:

```
<project-root>/
├── profile.md              ← interview answers + derived constraints
├── index.html              ← project-wide table of contents
├── BLUEPRINT.md            ← (optional) keep a copy of this file for reference
├── .ai/
│   └── skills/
│       ├── story/SKILL.md       ← writes pure target-language story.md
│       ├── enrich/SKILL.md      ← writes enrichment.toml
│       └── render/
│           ├── SKILL.md         ← guides the LLM to design the per-story page
│           └── render.py        ← helper: applies enrichment, refreshes project index
└── stories/
    └── NN-slug/             ← one folder per story
        ├── story.md         ← Spanish/French/Japanese/... body + frontmatter
        ├── enrichment.toml  ← per-word, per-sentence, per-phrase data
        └── index.html       ← bespoke designer-authored render (script wraps words in place)
```

### `profile.md` template

```markdown
# Reader profile

## Language pair
- Native language: <e.g. Russian>
- Target language: <e.g. Spanish>
- Variant: <e.g. neutral Latin American>

## Level
- CEFR: <e.g. A1>
- Self-description: <e.g. "knows ~500 words, can read very slowly">
- Words per text: <e.g. 150–300>
- New-vocab ratio: <e.g. 80% in-level / 20% above>

## Taste
- Topics that resonate: <bulleted list>
- Anti-vibes (do NOT include): <bulleted list>
- Reference works: <bulleted, with the specific resonant quality of each>
- Tone: <spectrum description>

## Shape
- Self-contained stories, no cliffhangers
- Shared universe: <one-paragraph world sketch, or "none">

## Format
- Grammar in popups: <rich | minimal>
- Wrap function words: <yes | no>
- Sentence translations: <off | opt-in toggle | always on>
- External fonts: <yes | no — if no, system fonts only>
- Dictionary URL pattern: <e.g. https://www.spanishdict.com/translate/{lemma}>

## Delivery
- Single HTML files on local disk, opened in browser
- No analytics, no trackers, no external dependencies
- (Google Fonts allowed only if `External fonts: yes` above)
```

### Project `index.html` template

A reader-friendly table of contents. Should feel intentional, not generic.
The baseline implementation uses a typewriter aesthetic (Courier Prime, cream
paper, dotted leaders) — pick whatever fits the reader's overall mood. This
page is the only surface that stays visually stable across stories; choose its
design once, then preserve it on every refresh.

Required structure:

- One `<a class="entry" href="stories/NN-slug/index.html">` per story.
- Each entry shows: zero-padded number, story title, short metadata (protagonist
  · setting), word count, date.
- A header strip with project name and a footer with a small motto/colophon.
- Entry count visible (`<span class="stat">03 entradas</span>` or equivalent in
  the target language).
- **Required machine markers** around the entries list — exactly these two HTML
  comments, on their own lines, with nothing else on the line:
  ```html
  <!-- STORIES:START -->
  <a class="entry" href="stories/01-...">...</a>
  <a class="entry" href="stories/02-...">...</a>
  <!-- STORIES:END -->
  ```
  These markers are the contract the render skill uses to find the insertion
  point. Everything between them is the entries list; everything outside is
  preserved untouched.

The project index is the only page that **persists across renders**. The
[render skill](#33-render-skill) refreshes it after every new story but preserves
the design and the entries already present.

---

## Part 3 — The three skills

These three skill files go in `.ai/skills/<name>/SKILL.md` inside the project.
They become invokable as `/<name>` (or via the agent's skill system) once the
project is active.

**If your agent has no skill system**, this still works without changes — treat
each `SKILL.md` as a self-contained prompt. To run a step, open the
corresponding `SKILL.md`, paste its body into the conversation as the
instructions for the next turn, then ask the agent to perform that step against
the project files. The three SKILL files below already include every input,
output, and rule the step needs to run in isolation.

The skills form a strict pipeline: **story → enrich → render**. Each consumes the
previous step's output. Skipping or reordering steps breaks the contract.

### 3.1 story skill

**Purpose**: write one new story in the target language, calibrated to
`profile.md`, with no annotations or translations in the visible text.

**Inputs**:
- `profile.md` (always read fresh)
- A one-sentence brief from the user (or the agent proposes 2–3 concepts and
  the user picks one)
- Existing `stories/NN-slug/story.md` files (for tone continuity and to avoid
  repeating premises)

**Output**: `stories/NN-slug/story.md` with frontmatter + body.

**Frontmatter schema** (YAML):

```yaml
---
slug: 03-zumbido-deriva       # NN-slug, zero-padded, ASCII-safe, target-lang slug
title: "Un zumbido en la deriva"  # in target language
date: 2026-05-11
words: 215                    # actual count, computed
protagonist: "Marco · navegante"
setting: "remolcador Tortuga · ruta a Ceres"
tone: "ironic realism with horror undertone"
references_to_other_stories: []   # for shared-universe continuity
new_keywords:                 # 5–10 above-level words to highlight
  - zumbido
  - deriva
  - sonda
---

<empty line>

<Body in target language only. Plain Markdown paragraphs separated by blank
lines. No bullet lists, no headings, no inline annotations, no English/native-
language anywhere in the body.>
```

**Body constraints**:

- Pure target language. No native-language glosses in the body, ever.
- Paragraphs separated by blank lines. No headings inside the body.
- Length matches `profile.md` (words-per-text).
- New-vocabulary ratio matches `profile.md`. The new words are the
  `new_keywords` from frontmatter, naturally embedded — not a vocabulary list.
- Style respects the references and anti-vibes from `profile.md`. If
  `profile.md` says "no moralizing", the story does not moralize.
- Self-contained unless the user explicitly asked for a serial.

#### Structural variety — give the renderer compositional material

A continuous wall of prose pushes the downstream `render` skill toward a
single centered column, because there is nothing else for it to lay out.
Where the story's content naturally allows it, **break the body into
structurally separable content blocks** so the renderer can distribute them
across the page as side panels, pull quotes, split-screen segments, framed
inserts, or marginalia.

This is **optional, not mandatory**. A tightly braided contemplative
monologue or a single confessional letter should stay continuous —
segmentation imposed on the wrong story damages it. But when the structure
permits, prefer outputs with naturally segmentable sections. Examples (mix
freely; the right two or three per story, not all of them):

- **Transmission logs** — timestamped signal entries, frequency or channel
  headers, sender/receiver fields, short bursts of dispatcher voice
- **Journal fragments** — dated diary or notebook entries, each self-
  contained, written in the protagonist's first-person register
- **Dialogue snippets** — short exchanges set apart from the surrounding
  narration (an interrogation, a radio call, an overheard scrap)
- **System messages** — terminal output, error codes, machine prompts,
  automated status lines in a flat machine voice
- **Flashback inserts** — short past-tense passages set off from the
  present-tense frame, marking a memory or earlier scene
- **Parallel scene fragments** — two or more scenes alternating in short
  blocks (ship + station, observer + observed, surface + orbit)
- **Observational side notes** — brief asides from the protagonist that
  read as marginalia or footnote-style commentary on the main action
- **Location/time jump separators** — labeled breaks between scenes (e.g.
  *— Día 412, 03:00, sala de control —*) that signal the renderer to start
  a fresh visual section

**How to mark segmentation in the .md** — use conventional Markdown
structure so the renderer can pick up the cues without bespoke parsing:

- `>` blockquote for transmissions, log entries, system messages, and
  quoted snippets
- A short level-2 heading (`## …`) or a labeled separator line for
  scene / location / time jumps
- `***` or `---` horizontal rule for major structural breaks between
  modular sections
- Plain paragraphs for the continuous narrative spine

Still pure target language. Still calibrated to `profile.md` (level,
new-vocab ratio, anti-vibes). Still no inline glossary, no native language,
no learner sidebars — **segmentation is a structural choice, not a
learner-aid leak**. Per-word and per-sentence translations are produced
later by `enrich`; the `.md` stays prose-only.

**Workflow**:

1. Read `profile.md`.
2. Scan `stories/` for existing story.md files (read frontmatter only) to keep
   tone consistent and avoid repeating premises.
3. If the user didn't specify a topic, suggest 2–3 concepts and let them pick.
4. Draft the story, count words, fill frontmatter.
5. Write to `stories/NN-slug/story.md`.
6. Report: the slug, the title, the word count, and a one-sentence pitch. Do not
   show the body unless asked — let the user read it in the rendered HTML.

**Numbering**: NN is the next zero-padded integer after the highest existing
story. Slug is in target language, transliterated to ASCII, lowercase,
hyphenated, ≤30 chars.

### 3.2 enrich skill

**Purpose**: produce `enrichment.toml` for one story — the data file consumed
by the renderer.

**Inputs**:
- `stories/NN-slug/story.md` (frontmatter + body)
- `profile.md` (for native language, dictionary URL pattern, grammar
  rich/minimal toggle, function-words toggle)

**Output**: `stories/NN-slug/enrichment.toml`.

**TOML schema**:

```toml
[meta]
slug = "03-zumbido-deriva"
title = "Un zumbido en la deriva"
target_lang = "es"
native_lang = "ru"

# Per-word entries. Key is the lowercased surface form as it appears in the
# story body. Each surface form gets exactly one [words."form"] table — TOML
# rejects duplicate keys, so you cannot stack multiple senses under the same
# form. For homographs (same form, different sense in different sentences),
# write a single entry with the gloss that best fits this story; if a specific
# occurrence needs a different reading, add a [[word_overrides]] entry below.
#
# tr      : translation into native language (1–5 words, concise)
# pos     : part of speech (noun | verb | adj | adv | pron | det | prep | conj
#                          | num | interj | proper). Use universal categories,
#                          not target-language-specific terminology. `proper`
#                          is reserved for proper nouns inside [[phrases]]
#                          entries — bare proper nouns in the body are never
#                          wrapped (see interview Q14).
# lemma   : dictionary headword. For nouns, include the article if the target
#           language uses gendered articles (e.g. "la deriva", not "deriva").
#           For verbs, the infinitive. For adjectives, the base form.
# grammar : OPTIONAL. Multi-line string in the grammar mini-language (see below).
#           Only fill this when grammar is non-obvious for the learner's level.

[words."trabaja"]
tr = "(он) работает"
pos = "verb"
lemma = "trabajar"
grammar = """
**trabaja** — *presente*, 3 л. ед. ч., от **trabajar** ("работать").

Правильный глагол на `-ar`: 3 л. ед. → окончание `-a`.

`trabaj·[[ar]]` → `trabaj·[[a]]`
"""

[words."deriva"]
tr = "дрейф"
pos = "noun"
lemma = "la deriva"

# Per-sentence translations. Sentences are listed in the order they appear in
# the body. The agent splits the body into sentences (target-language-aware
# punctuation: ., !, ?, ¡, ¿, 。, ！, ？ — pick what applies).
#
# Used by the renderer to optionally show full-sentence translations on a
# secondary hover or click affordance (the design contract leaves this
# optional; A1 readers benefit, B2+ readers may want it off).

[[sentences]]
es = "El zumbido empezó el martes."
tr = "Гудение началось во вторник."

[[sentences]]
es = "Marco trabaja solo en el remolcador."
tr = "Марко работает один на буксире."

# OPTIONAL: per-occurrence overrides for homographs. Only fill these when the
# default [words."form"] gloss is actively misleading in a specific sentence.
# `sentence` is the 1-based index into [[sentences]] above. `occurrence` is the
# 1-based index of this surface form within that sentence (default 1). The
# renderer prefers an override whose (form, sentence, occurrence) matches the
# current word; otherwise it falls back to [words."form"].

# [[word_overrides]]
# form = "banco"
# sentence = 4
# occurrence = 1
# tr = "скамейка"          # default [words."banco"] said "банк" — wrong here
# grammar = """..."""      # optional, same mini-language as [words.*]

# Per-phrase entries. Use for multi-word units that aren't compositionally
# obvious from their parts: idioms, proper-noun compounds, fixed expressions.
# These are matched as longest-match-wins by the renderer when wrapping words,
# so a phrase like "Pioneer-12" is wrapped as a single span.

[[phrases]]
form = "Pioneer-12"
tr = "Пионер-12 (зонд НАСА, 2031)"
pos = "proper"
note = "fictional NASA probe, launched 2031, the silent encounter in this story"
```

#### TOML key quoting

TOML bare keys allow only `A-Z a-z 0-9 _ -`. Surface forms that contain anything else — accents (`á é í ó ú`), `ñ`, `ü`, cedillas, the French elided `'`, CJK glyphs, anything non-ASCII — **must** be wrapped in double quotes in the table header. Without quotes, `tomllib` (and any spec-compliant TOML parser) rejects the file with a misleading `Expected ']' at the end of a table declaration` error pointing at the offending line.

```toml
# ✗ wrong — will fail at parse time
[words.señal]
[words.être]
[words.猫]

# ✓ correct — quoted keys
[words."señal"]
[words."être"]
[words."猫"]
```

The two forms are equivalent for lookup (`words.get(form.lower())` works for either) — the quoting is purely a TOML-syntax requirement. When in doubt, quote: ASCII-only keys tolerate quotes too, so it's safe to standardize on always-quoted. The example schema above already follows this pattern (`[words."trabaja"]`); make it habit for every entry.

The helper script (see [render skill](#33-render-skill)) wraps the TOML load in a `try/except`: when it sees this error class it surfaces a targeted hint naming the offending line and suggesting the quoted form, so the failure is self-correcting on the next run.

#### Grammar mini-language

Used inside `grammar = """..."""` strings. Parsed by the renderer's inline
`mdToHtml()` (see [render skill](#33-render-skill)) into HTML at popup-show time.

| Syntax | Renders as | Use for |
|--------|------------|---------|
| `**foo**` | `<b>foo</b>` | The current surface form, emphasized headwords |
| `*foo*` | `<i>foo</i>` | Grammatical category names, technical terms |
| `` `foo` `` | `<code>foo</code>` | Morphological breakdowns, paradigm cells |
| `[[suffix]]` | `<u>suffix</u>` | Inside or outside code — highlights the morphological piece that changed |
| Blank line | new paragraph | Paragraph separation |

Example:

```
**hablo** — *presente*, 1 sg., от **hablar** ("говорить").

Правильный глагол на `-ar`: 1 л. ед. → окончание `-o`.

`habl·[[ar]]` → `habl·[[o]]`
```

Rules:

- Keep grammar blocks short. Two or three paragraphs maximum. Anyone needing
  more goes to the dictionary link.
- Write grammar in the **native** language, not the target.
- Don't repeat the translation — the popup already shows `tr` separately.
- Highlight the morphological piece that distinguishes this form from the
  lemma. The learner's eye should land on `[[o]]` and understand "ah, the `-o`
  suffix marks 1sg present".

**Workflow**:

1. Read `story.md` and `profile.md`.
2. Tokenize the body into words. Respect target-language tokenization rules
   (spaces for Latin/Cyrillic/Greek; segmentation for CJK; particle awareness
   for agglutinative languages). For non-segmented scripts, the agent uses its
   own judgment about what counts as a word — pick the level of granularity
   that's most useful to a learner.
3. For each unique surface form, write a `[words."form"]` entry. Translate
   into the native language. Tag POS using the universal categories above.
   Determine the lemma.
4. If `profile.md` says **rich grammar**, add `grammar = """..."""` for any
   form where the morphology is non-obvious to a learner at this level. Skip
   it for transparent regulars at higher levels.
5. Split the body into sentences. Translate each one as a `[[sentences]]`
   entry.
6. Identify multi-word phrases that need single-popup treatment. Add them as
   `[[phrases]]`.
7. Write to `stories/NN-slug/enrichment.toml`.

**Quality bar**:

- Translations are concise (1–5 native-language words). Long definitions belong
  in the dictionary link, not the popup.
- POS is one of the universal categories — don't invent target-language-specific
  POS tags here.
- The lemma is the dictionary entry the learner would search for, not an
  abstract linguistic root. For Spanish nouns, include the gendered article.
  For Japanese verbs, the dictionary form. For German nouns, capitalized and
  with article.
- Every word in the body that the user might mouse over is covered. The
  renderer falls back to nothing if a word isn't in the TOML, which is a bad
  UX — coverage should be near 100%.

### 3.3 render skill

**Purpose**: produce `stories/NN-slug/index.html` — a hand-designed,
self-contained HTML page implementing the hover-popup vocabulary system. Then
refresh the project-wide `index.html` to add an entry for the new story.

**The split — design vs. mechanical wiring**

This skill is delivered in **two pieces** because the work splits cleanly:

- The **designer** (LLM) writes the HTML page freely — palette, typography,
  layout, illustration, popup look. The only contract is one attribute on the
  element wrapping the story body: `data-story-body`.
- The **helper script** (`.ai/skills/render/render.py`) walks inside that
  element, wraps every word and sentence-terminator with the right `data-*`
  spans using `enrichment.toml`, and auto-injects two managed blocks: popup
  CSS invariants in `<head>`, popup JS before `</body>`. Re-runnable.

The bespoke design choices that make a story page distinctive are LLM
territory. The repetitive, easy-to-get-wrong-by-hand wiring (escaping,
sentence pairing, popup behavior invariants) is the script's job. Letting the
script handle the latter frees the designer to push the design without
boilerplate fatigue.

**Inputs**:
- `stories/NN-slug/story.md`
- `stories/NN-slug/enrichment.toml`
- `profile.md`
- The existing project `index.html` (the script preserves it and refreshes only the entries list)
- The helper script at `.ai/skills/render/render.py`

**Output**:
- `stories/NN-slug/index.html` (designer writes the page; script applies enrichment in place)
- `index.html` (auto-updated by the script)

This skill has two contracts: **functional** (the popup system must work the
same across all stories — enforced by the script) and **design** (every story
must look different — the LLM's job).

#### Designer's contract

The `index.html` file is a complete self-contained HTML document. The designer
writes it however they want. The only required element:

```html
<article data-story-body>
  <p>{paragraph 1 of story.md, verbatim}</p>
  <p>{paragraph 2 of story.md, verbatim}</p>
  ...
</article>
```

- The Spanish/French/Japanese/... text inside `<p>` tags matches the body of
  `story.md` exactly, paragraph for paragraph.
- The wrapping element can be `<article>`, `<div>`, `<section>` — anything;
  the script keys off the `data-story-body` attribute.
- HTML inside paragraphs is preserved on tokenization (you can use `<em>`,
  `<small>`, etc. — only text between tags is wrapped).
- Everything else in the document is the designer's choice: structure,
  styles, fonts, hero SVG, popup look.

The script auto-manages two blocks (the designer **must not** hand-edit them
— each script run replaces them):

- `<style data-popup-invariants>…</style>` at end of `<head>` —
  behavior-critical popup CSS (pointer-events, hit-area bridge,
  transitions), plus the reading-progress hairline styling and the
  `prefers-reduced-motion` guard. See
  [Part 4 invariants](#part-4--critical-invariants) for the specific rules
  and why they're locked.
- `<script data-popup>…</script>` before `</body>` — popup
  show/hide/position logic, the grammar parser `mdToHtml()`, and the
  reading-progress scroll listener (which appends a
  `<div class="reading-progress">` to body on load).

Designer styles **everything visual** via their own `<style>` block (declared
above the auto-managed block in source order — designer rules win because of
the cascade). Classes the designer is expected to style:

- `.w` and `.s` — word and sentence-terminator hover triggers (the script's
  invariants set only `cursor` and a default `border-bottom: 1px dotted
  currentColor`; designer can override base color, hover color, transition).
- `.pop` — the floating tooltip container
- `.pop.has-grammar` — wider variant when a grammar block is present
- `.pop.sentence` — variant for sentence-terminator hover
- `.pop .lemma`, `.pop .pos`, `.pop .tr` — popup contents in order
- `.pop .g-block` and its descendants `b` / `i` / `code` / `u` — grammar block
  rendered from the mini-language
- `.pop .link` — dictionary link
- `.pop.sentence .s-label`, `.pop.sentence .s-tr` — sentence popup parts

To re-color word/sentence underlines on hover, use higher specificity
(e.g. `article .w:hover`) since the auto-managed block uses `currentColor`.

#### Helper script

The reference implementation lives at `.ai/skills/render/render.py` in the
project. It's ~500 lines of stdlib Python (3.11+, uses `tomllib`). Any
language works — Node with a TOML dep, Bun, Deno — as long as the script
honors the contract above.

```bash
# from project root
py .ai/skills/render/render.py                                # auto-pick newest story; apply enrichment to existing index.html
py .ai/skills/render/render.py stories/05-pasajero-unico
py .ai/skills/render/render.py --bootstrap stories/06-foo     # write a minimal design scaffold for a new story
py .ai/skills/render/render.py --bootstrap --force stories/05 # overwrite an existing index.html with a scaffold (destructive)
py .ai/skills/render/render.py --index-only                   # only refresh the project-wide index.html
```

Reports sentences paired and any words missing from `[words.*]` so the user
can extend the enrichment and re-run.

**Re-runnability**: each run **strips prior `.w` / `.s` spans first**, then
re-tokenizes. The designer can edit the Spanish text inside `<p>` tags freely
between runs — the script picks up the edits. (Keep the body in sync with
`story.md` so `enrich` re-runs aren't surprised.)

**Bootstrap mode**: when starting a new story, `--bootstrap` writes a minimal
HTML scaffold containing the body marker and the story paragraphs as plain
`<p>` tags with default styling. The designer then redesigns from there.

#### Functional contract — same for every story

The script enforces this. The shape below is what the script emits; the
designer doesn't write it by hand.

##### Word wrapping

Every meaningful word in the body is wrapped:

```html
<span class="w"
      data-tr="brief native-language gloss"
      data-pos="noun|verb|adj|adv|...|proper"
      data-lemma="dictionary form"
      data-grammar="**form** — *category*&#10;&#10;...">word</span>

<span class="s" data-tr="full sentence translation">.</span>
```

- `data-grammar` is OPTIONAL and only present if the TOML had a `grammar` field.
- Newlines inside `data-grammar` are encoded as `&#10;` (HTML numeric entity).
  Double quotes inside any attribute are encoded as `&quot;`.
- Punctuation, numerals, and proper nouns stay **outside** `.w` spans
  (unless flagged as a `[[phrases]]` entry, in which case the phrase becomes one
  span).
- Sentence-terminators (`.?!`) get a `.s` span paired in document order with
  the corresponding `[[sentences]].tr` entry.

##### Popup behavior

A single `<div class="pop">` is mounted once at the document root and reused
for every hover/tap. Positioned absolutely under JS control.

- **Desktop**: `mouseenter` on a `.w` shows the popup; `mouseleave` (after a
  short delay if the cursor enters the popup itself) hides it.
- **Touch / mobile**: tap shows; tap elsewhere hides; second tap on the same
  word can navigate to the dictionary link.
- The popup contains, in order:
  1. **Lemma** in bold (display font).
  2. **Translation** (native language).
  3. **Part of speech** in small caps or italic, muted.
  4. **Grammar block** rendered from `data-grammar` via `mdToHtml()` — only if
     present.
  5. **Dictionary link** at the bottom: text like "Open in <Dict> ↗" or the
     native-language equivalent.
- Positioning: above the word by default. Flip below if within ~120px of the
  viewport top. Keep within horizontal viewport bounds (clamp left/right).

##### `mdToHtml()` reference implementation

Paste this verbatim into every render's inline `<script>`. Do not extend or
generalize it — it's intentionally tiny and language-agnostic.

```js
function mdToHtml(md){
  if(!md) return '';
  md = md.replace(/&/g,'&amp;').replace(/</g,'&lt;').replace(/>/g,'&gt;');
  return md.split(/\n\n+/).map(function(p){
    p = p.replace(/`([^`]+)`/g, function(_, c){
      return '<code>' + c.replace(/\[\[([^\]]+)\]\]/g, '<u>$1</u>') + '</code>';
    });
    p = p.replace(/\*\*([^*]+)\*\*/g, '<b>$1</b>');
    p = p.replace(/\*([^*]+)\*/g, '<i>$1</i>');
    p = p.replace(/\[\[([^\]]+)\]\]/g, '<u>$1</u>');
    return '<p>' + p + '</p>';
  }).join('');
}
```

Notes:

- HTML-escapes raw `&`, `<`, `>` first, so attribute content is safe to inject.
- `[[suffix]]` works both inside and outside `` `code` `` (the inner regex
  inside the code-replacer handles the nested case).
- This intentionally does not support lists, links, headings, or block code —
  the popup grammar block is small and tight.

##### Dictionary link

Use the `dictionary URL pattern` from `profile.md`. Substitute `{lemma}` with
the URL-encoded lemma at render time:

```js
const dictUrl = lemma => pattern.replace("{lemma}", encodeURIComponent(lemma));
```

The pattern is whatever the user picked (e.g.
`https://www.spanishdict.com/translate/{lemma}`).

##### Sentence translations

Behavior is driven by `Sentence translations:` in `profile.md`:

- **off** — do not surface them at all. The `[[sentences]]` data stays in the
  TOML, unused by this page.
- **opt-in toggle** — render a small chrome button (e.g. "español ↔ español + tr").
  Clicking reveals the native-language translation inline beneath each sentence.
  Default state per page load is hidden.
- **always on** — render each sentence with its translation directly beneath,
  styled subtly (smaller, muted) so it doesn't dominate the target text.

Source the translations from `[[sentences]]` in the TOML.

#### Found-document framing

Every render is treated as **a specific in-universe artifact**, not "a story page about X". The page IS the document the protagonist would produce, receive, or be cataloged into — a found object the reader has stumbled on. This is the primary discipline that forces real variety; without it, even careful palette/font work drifts toward "yet another centered serif column on a dark bg".

Each story has at least one natural artifact to pick from. Examples by protagonist:

| Protagonist | Natural artifact |
|-------------|------------------|
| AI / observatory | terminal log, telemetry readout, system console output |
| Anthropologist / observer | field journal, ethnographic notes, expedition diary |
| Navigator / pilot | ship's bridge log, course-correction sheet, bridge HUD |
| Biologist | lab notebook, specimen plate, sample-jar label sheet |
| Linguist / translator | translation worksheet, gloss table, parallel manuscript |
| Old protagonist / chronicler | handwritten letter, memorial placard, chronicle entry |
| Engineer / mechanic | maintenance log, parts diagram, wiring schematic |
| Astronomer / radio operator | observation log, signal trace plot, deep-survey sheet |
| Cargo / customs / port officer | bill of lading, customs declaration, manifest |
| Programmer / sysadmin | source file with comments, ticket queue, runbook |
| Doctor / medic | medical chart, patient form, triage sheet |
| Gardener / archivist | herbarium card, accession ledger, index card |

The framing shapes **every** decision: title becomes a document header, metadata becomes the document's metadata (file number, paper stock, redactions, stamps), the prose lives where it would live on that artifact (numbered log entries, journal pages, form fields, gridded notebook), and decorations are the artifact's natural marks (timestamps, signatures, perforations, ink bleeds, registration crosses, censor bars). The target-language body still reads as continuous prose, but is *housed* inside the artifact's structure.

Anti-pattern: the protagonist's profession leaks into a few decorative SVGs but the underlying page is still "centered serif column". That fails this discipline.

#### Format archetypes — variety rule

Pick the artifact's format from the named-archetype list below. The list is illustrative, not exhaustive — invent new archetypes when the story warrants. **Anti-repeat rule**: no archetype reused within 3 consecutive stories. After three stories away, an archetype may return, but the second take must be a visibly fresh interpretation (different palette, different layout primitive, different motion family) — never a re-skin.

Named archetypes (mix freely with new ones):

- **Terminal / system log** — monospace amber-on-black or green-on-black, timestamped lines, blinking cursor, ANSI box-drawing chrome
- **Manuscript folio** — parchment, two-column with marginalia, illuminated drop cap, calligraphic display, rubricated initials
- **Observation log / lab notebook** — gridded or ruled paper, handwriting body, monospace headers, plotted traces, marginal scribbles
- **Customs form / port manifest** — bureaucratic grid, stamped fields, typewriter monospace, carbon-copy ink, official seal corner
- **Telegram tape** — narrow column, perforation strip, ALL CAPS, STOP between sentences, ribbon-typewriter ink
- **Magazine / newsprint spread** — multi-column halftone, era-specific display face, byline strip, pull-quote
- **Museum placard / wall text** — gallery serif, paired columns, generous whitespace, brass-plaque header, plain background
- **Bridge HUD / instrument readout** — vector lines, faux-radar, scanlines, sci-fi sans, telemetry numerals
- **Handwritten letter** — paper texture, handwriting body, return-address block, postmark, signature flourish
- **Field journal** — leather-board frame, pressed-plant illustrations, dated entries, ink-bleed, weathered paper
- **Index card / library catalog card** — 3×5 ruled card, monospace, hand-stamped accession number
- **Postcard** — illustration top, prose-on-the-back layout, stamp + postmark
- **Parish chronicle / annal** — heavy display caps for the year, single-column with rubricated marks, ecclesiastical serif
- **Engineering schematic** — line-art, callouts, parts list along the margin, drafting-style sans
- **Source file with comments** — IDE chrome, line numbers, syntax-coloured prologue/epilogue, the story housed inside a docstring/comment block, status bar, blinking cursor

The archetype determines: page chrome and "edges" (paper, console frame, card border), typography family, palette discipline (paper inks vs. console phosphors vs. plate-printed CMYK), layout primitive (single column / two-column / framed / gridded / freeform marginalia), motion sensibility (paper rustle vs. scanline drift vs. ink bleed vs. dial sway), and **what the metadata strip becomes** (file number, accession, dispatch number, frequency band, postal stamp).

Keep a running mental note of the prior three stories' archetypes when planning a new render. If unsure, scan recent `stories/*/index.html` for their archetype before committing to one.

#### Motion — auto-injected and designer-authored

Two motion bits are auto-managed by the helper script (designers don't write them; they theme or hide them):

- **Reading-progress hairline** — a `<div class="reading-progress">` is appended to `<body>` by the popup script. Its width tracks `scrollTop / (scrollHeight − clientHeight)`. CSS lives in the `<style data-popup-invariants>` block. The bar reads its color from the CSS variable `--accent` on `:root` or `body` (fallback `currentColor`). Each story declares `:root { --accent: <story-color>; }` so the bar matches the palette. Hide per-page with `.reading-progress { display: none; }` if it doesn't fit the design — but prefer themed over hidden.
- **`prefers-reduced-motion` guard** — a media query in the same invariants block short-circuits all CSS animations and transitions and hides the reading bar when the user has opted out. Designer-authored motion must rely on standard CSS `animation` / `transition` declarations so this guard takes effect. If JS-driven motion is unavoidable, gate it on `window.matchMedia('(prefers-reduced-motion: reduce)').matches`.

Designer-authored motion (optional, per-story):

- **Ambient background motion** — a slow, low-opacity layer behind the text that reinforces mood: drifting starfield for cold sci-fi, slow conic-gradient drift for manuscript-warm, faint flicker/scanlines for hard sci-fi, ink-bleed wash for contemplative, paper-rustle for handwritten formats. Multi-second cycles, behind text, never animating the body text itself. Pure CSS keyframes or inline animated SVG — **no raster GIFs**, no external image URLs.
- **Refined popup entrance** — override the default `.pop` / `.pop.show` transition with a story-themed reveal (filter blur → clear, small `translateY`, soft `scale(0.96) → 1`, ink-bleed). The behavior invariants (`opacity`, `pointer-events`, the `::after` bridge) must stay intact — enrich, don't replace. The `.w:hover` underline can also animate (e.g. a gradient `background-size` growing from 0 to 100%) instead of the static dotted line.

Motion is decorative — every animation remains readable, slow, and quiet.

#### Design contract — different for every story

Every render must invent design choices driven by the story's frontmatter and
content. Same functional behavior, completely different visual identity.

- **Color palette**: 3–6 colors derived from the setting and mood. Examples:
  - Saturn observatory (cold, contemplative) → deep navy, silver-ice, faint gold
    accents on rings
  - Desert colony (sun-bleached, archaic) → terracotta, ochre, gold leaf, dark wine
  - Kuiper Belt (icy, ironic, lonely) → near-black void, harsh cyan, dirty white,
    hazard-orange accent
- **Typography**: two paired families — one display, one body. Never reuse the
  same pair across stories; the agent maintains a tally and picks a new combo
  each time. The source depends on `External fonts:` in `profile.md`:
  - **External fonts = yes** — Google Fonts. Examples:
    - Medieval, religious: *Cinzel* + *EB Garamond*
    - Hard sci-fi: *Space Grotesk* + *JetBrains Mono*
    - Contemplative literary: *Cormorant Garamond* + *Inter*
    - Industrial / hazard: *Major Mono Display* + *Space Grotesk*
  - **External fonts = no** — system stacks only. Skip the Google Fonts
    `<link>` entirely. Build pairs from broadly available families, e.g.
    serif body (`Georgia, "Iowan Old Style", "Palatino Linotype", serif`)
    paired with mono display (`"SF Mono", Menlo, Consolas, monospace`), or
    sans body (`-apple-system, "Segoe UI", system-ui, sans-serif`) paired
    with serif display (`"Iowan Old Style", Georgia, serif`). Vary mood by
    rotating which side is display vs body.
- **Hero illustration**: hand-coded inline SVG (50–300 lines) thematic to the
  story. Abstract or representational, stylized, never photorealistic. Original
  per render — no copy-paste from previous stories.
- **Background texture**: subtle gradients, blur, SVG-filter noise, conic
  patterns, scanlines, vignettes — pick what fits. Atmospheric, not loud.
- **Layout / composition**: vary aggressively across stories. **Avoid
  defaulting to a single narrow horizontally centered text column** — that is
  the AI-design comfort zone and reads as generic across stories. Reserve the
  centered-column choice for stories that explicitly benefit from minimalism
  (intimate monologue, brief confessional letter, museum-placard quiet).
  Otherwise, reach for editorial, cinematic, atmospheric, spatially expressive
  compositions: asymmetric grids, side-margin metadata, faux-interface frames
  (log-entry consoles, letter paper, hazard documents), split-screen panes,
  multi-column excerpts. The composition is part of the storytelling. See
  [Layout philosophy](#layout-philosophy--editorial-cinematic-spatial) below
  for the full toolkit and constraints.
- **Reading width**: ~640–720px max for body text. Never less than 18px font
  size. Line-height 1.7–1.9 for low-CEFR readers, 1.6–1.7 for higher.

The page should feel like the interior of a book imagined for **this specific
story**, not a generic learner-app template.

#### Layout philosophy — editorial, cinematic, spatial

Treat the page as a designed spread, not a text dump piped into a column. The
renderer should compose space deliberately — text, illustration, framing
chrome, and ambient layers arranged like an editorial print spread or a
genre-specific publication, while remaining fully static HTML/CSS. The narrow
centered column is the path of least resistance and the easiest tell of
generic AI design; only choose it when minimalism is the right answer for
*this* story.

**Permitted and encouraged techniques** (mix freely — most pages combine 3–5):

- **Asymmetrical layouts** — text weighted off-axis, grids that break neatly,
  alignment shifted left or right rather than centered around the page midline
- **Offset or floating text regions** — paragraph blocks at varied indents,
  callout blocks pulled into the margin, hanging headings, prose that wraps
  around illustration or framing chrome
- **Side annotations or contextual panels** — protagonist's marginalia,
  parallel-column footnotes, paratextual stamps and labels along an edge,
  datasheet metadata in a sidebar
- **Pull quotes and highlighted fragments** — a single sentence enlarged in a
  contrasting face, framed by hairlines, color blocks, or generous negative
  space
- **Split-screen compositions** — two-pane layouts where one side carries
  prose and the other carries inline SVG, telemetry, a portrait, or
  contrasting type
- **Layered gradients and atmospheric backgrounds** — multiple gradient stops,
  conic + radial + linear combined, behind faint SVG noise, vignettes, or
  filter blurs
- **Modular sections with distinct visual tone** — different blocks of the
  story styled differently (journal section vs. console section vs. flashback
  section on the same page), each with its own micro-treatment
- **Controlled whitespace and experimental spacing** — generous gutters
  between blocks, unconventional line-lengths per region, intentional
  "negative space" passages between scenes
- **Multi-column excerpts** — newspaper-style or manuscript-style multi-column
  passages for specific blocks (not necessarily the whole story)
- **Embedded "terminal", "transmission", or "log" blocks** — monospace
  box-drawn frames, timestamped lines, signal-trace panels, console output
  dropped inline among the prose
- **Visual framing inspired by futuristic interfaces or genre publications** —
  HUD chrome, dashboard panels, magazine masthead, editorial drop caps,
  datasheet headers, signal-trace strips

These techniques **compose**: a page can be a split-screen with a
side-annotation column on the prose pane and an embedded transmission log on
the other; or an asymmetric editorial spread with a pull quote, a multi-column
lower section, and a layered gradient under everything. Lean into the story's
content — if the source `.md` already segments into logs, journal entries, or
scene fragments (see the [story skill's structural variety
note](#structural-variety--give-the-renderer-compositional-material)),
distribute those across the page rather than concatenating them into one
column.

**Constraints — non-negotiable**:

- **Readability has priority.** Body text never drops below 18 px,
  line-height in the range set by the Reading-width bullet above, contrast
  ratio ≥ 4.5:1 against its background. Every decorative pass must improve
  the page; if a flourish hurts reading, cut it.
- **Layouts must remain responsive.** Use fluid units (`clamp()`,
  `minmax()`, percentages, `vw/vh`), CSS Grid and Flexbox. Multi-column,
  split-screen, and floating regions must collapse to a clean single column
  on narrow viewports — no horizontal scroll, no overlapping text, no
  truncated panels on mobile.
- **Semantic HTML stays intact.** Story body still lives inside
  `<article data-story-body>` with real `<p>`, `<h1>`, `<h2>`,
  `<blockquote>`, `<aside>`, `<figure>` tags. Don't replace paragraphs with
  `<div>` salad to hit a visual; don't strip headings for compactness; don't
  break the structure the tokenizer depends on (paragraphs of text inside
  the `data-story-body` element).
- **Contrast and typography stay accessible.** Real font sizes, real
  line-height, real color contrast — including inside framed panels,
  marginalia, and split panes. Decorative scripts or display faces belong
  on titles, pull quotes, and metadata strips — never on the body prose.
- **Decorative effects never overpower the story text.** Ambient gradients,
  framing chrome, marginalia, and split-pane visuals stay quieter than the
  prose. If the eye lands on the decoration before the words, dial it back.
  Pull quotes and highlights are the one allowed exception, and only
  because they *are* story text.
- **Motion is avoided unless explicitly requested.** Default to fully
  static. The ambient-motion options described elsewhere in this contract
  are opt-in, not default; they apply when the story or the user clearly
  calls for atmosphere. When motion is used, follow the existing motion
  rules (slow, low-opacity, behind text, `prefers-reduced-motion`
  respected, no raster GIFs).

#### Project-index refresh

After applying enrichment, the script **also** updates the project
`index.html`:

- Locates the `<!-- STORIES:START -->` / `<!-- STORIES:END -->` markers. If
  either is missing, **stops with an error** rather than guessing — the
  markers are the contract; without them, "the entries list" isn't safely
  defined.
- Regenerates the entries list from the story frontmatter on disk
  (enumerates every `stories/NN-slug/` folder and reads its `story.md`).
- Updates the entry-count badge (e.g. `03 entradas` → `04 entradas`).
- Preserves everything outside the markers byte-for-byte — header, footer,
  styles, scripts, design. This file is a stable artifact, not a per-render
  bespoke surface.

#### Back-link

Each story page has a small back-link to the project index. Path is
`../../index.html` from `stories/NN-slug/index.html`. The designer includes
this in the page (it's not auto-injected).

#### Workflow

1. Read `story.md`, `enrichment.toml`, `profile.md`.
2. **Pick the found-document framing.** Identify what in-universe artifact
   this story should *be* (see [Found-document framing](#found-document-framing)):
   the protagonist's log, a letter, a customs form, an observation sheet,
   a manuscript page, a source file, etc. Lock the artifact before any
   visual decisions.
3. **Pick the format archetype** from the named list (or invent a fresh
   one). Check the prior 3 `stories/*/index.html` for their archetype — do
   not reuse any of them. Record the archetype choice.
4. Decide design direction *within the chosen archetype's idiom* —
   palette, font pairing, page chrome, marginalia, motion family. The
   choices serve the artifact, not generic "story page" aesthetics.
   Briefly — internal, not shown to user.
5. **Design the page**: write `stories/NN-slug/index.html` from scratch,
   or run `py .ai/skills/render/render.py --bootstrap stories/NN-slug`
   for a minimal scaffold to start from. The target-language text goes
   inside an element marked `data-story-body`; everything else (palette,
   fonts, hero SVG, popup look) is the designer's choice. The designer
   styles `.w`, `.s`, `.pop`, `.lemma`, `.pos`, `.tr`, `.g-block`,
   `.s-label`, `.s-tr`, `.link` via their own `<style>` block. Declare
   `:root { --accent: <color>; }` so the auto-injected reading-progress
   bar picks up the palette.
6. **Apply enrichment**: run `py .ai/skills/render/render.py stories/NN-slug`.
   The script wraps every word and sentence-terminator inside
   `[data-story-body]`, injects the popup CSS invariants and the popup
   JS, and refreshes the project-wide `index.html`. If it reports missed
   words, extend `enrichment.toml` and re-run (the script picks up the
   changes).
7. Report to user: file path, **artifact + archetype** picked,
   one-sentence thematic summary (palette, font pair, motif, motion
   family), missed words if any, and a hint command to open in browser.

---

## Part 4 — Critical invariants

These are bugs already paid for in the baseline implementation. Don't repeat them.

> Invariants 1 and 2 are auto-managed by the render helper script — it injects
> them as `<style data-popup-invariants>` at the end of `<head>` on every run.
> The designer should not duplicate or override these specific rules. The
> invariants are still listed here because anyone porting the script to a new
> language must implement them, and anyone debugging "popups flicker" should
> check the auto-injected block first.

### Invariant 1 — Popup pointer-events when hidden

The hover popup `<div class="pop">` is mounted once at document level and
positioned absolutely over the body. When **hidden**, it must NOT block pointer
events on the words underneath, or it will silently swallow `mouseenter` on
any word it happens to be sitting over from its previous show.

```css
.pop {
  pointer-events: none;   /* default: invisible AND non-blocking */
  opacity: 0;
}
.pop.show {
  pointer-events: auto;   /* only intercept events while shown */
  opacity: 1;
}
```

**Symptom if missing**: "Sometimes when I hover a word, no popup appears."
Intermittent because it depends on where the previous popup was positioned.

### Invariant 2 — Hover bridge size

A `.pop::after` "bridge" sits between the word and the popup so the cursor can
travel into the popup without crossing a gap that triggers `mouseleave`. Set
its height to ~12px, not 18px.

```css
.pop::after {
  content: '';
  position: absolute;
  bottom: -12px;     /* connects to word above */
  left: -12px;
  right: -12px;
  height: 12px;
}
.pop.below::after { bottom: auto; top: -12px; }   /* flip when popup renders below */
```

**Symptom if too tall**: the bridge overlaps the line of words **below** the
popup's word, blocking their mouseenter events.

### Invariant 3 — Data-driven grammar, not auto-generated

Don't build a generic JS engine that inspects `data-pos` + `data-lemma` + word
text and synthesizes grammar HTML on the fly. It seems clever but grows
linearly with new stories: every irregular verb, every stem change, every
pronominal quirk requires another hardcoded case. Curate grammar **per word**
in `enrichment.toml`, parse it with a tiny generic `mdToHtml()`, and move on.

### Invariant 4 — One bespoke design per story

No shared base HTML template across stories. Each story is its own
hand-designed artifact. The shared parts are the **functional contract**
(popup behavior, CSS invariants, mdToHtml()), not the visual layer. Pasting the
same base CSS across stories produces the "AI learning app" aesthetic the
project is explicitly trying to avoid.

### Invariant 5 — Single self-contained HTML file

No external JS frameworks, no jQuery, no Tailwind CDN, no Bootstrap, no
analytics, no Google Tag Manager. The only external dependency permitted is
Google Fonts, and only if the user opted into `External fonts: yes` in
`profile.md`. Everything else — CSS, JS, illustrations — is inline. The page
must remain functional offline (degrading to system fonts if Google Fonts is
loaded but unreachable).

### Invariant 6 — Pure target language in the visible body

The story body shows the target language only. Translations live in popups,
sentence translations live in an opt-in mode, full-text English/native-language
"side-by-side" views are explicitly out of scope. If the user wants
side-by-side, this isn't the right tool — but they almost never do, once they
try the popup model.

### Invariant 7 — Slugs are stable

A story's `NN-slug` is its identity. Don't rename slugs after publication; it
breaks any bookmarks the user may have made. If a typo needs fixing, fix it in
the `title` frontmatter field, not the slug.

### Invariant 8 — Reading-progress hairline + reduced-motion guard

The helper script auto-injects a reading-progress `<div class="reading-progress">` into `<body>` on load and a scroll listener that tracks `scrollTop / (scrollHeight − clientHeight)`. The CSS sits in the same `<style data-popup-invariants>` block as the other popup invariants and is themable via `--accent`. The same block carries a `@media (prefers-reduced-motion: reduce)` guard that short-circuits all CSS animations and transitions and hides the progress bar.

```css
.reading-progress {
  position: fixed; top: 0; left: 0; height: 2px; width: 0;
  background: var(--accent, currentColor);
  z-index: 1001; pointer-events: none;
  transition: width 0.08s linear;
}
@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after {
    animation-duration: 0.001ms !important;
    animation-iteration-count: 1 !important;
    transition-duration: 0.001ms !important;
  }
  .reading-progress { display: none; }
}
```

Designer-authored motion (ambient background layers, popup entrance animations, hover-underline draws, SVG keyframes) must use standard CSS `animation` / `transition` declarations rather than JS-driven `requestAnimationFrame` loops so this global guard actually applies. If JS-driven motion is unavoidable, gate it on `window.matchMedia('(prefers-reduced-motion: reduce)').matches`.

**Symptom if missing**: progress bar visible to users who opted out of motion; ambient animations keep running for vestibular-sensitive users. Accessibility regression that won't show up in casual QA.

### Invariant 9 — Word-boundary edge cases

- Apostrophes in target languages (French *l'eau*, English *don't*, Italian
  *l'arte*) — these are often two grammatical words inside one
  orthographic-looking unit. Tokenize at the apostrophe; both halves get their
  own `.w` wrapper.
- Hyphenated compounds (Spanish *Pioneer-12*, German *Klassenkampf-Romantik*)
  — usually one unit, but check the lemma. Default to one span per orthographic
  word.
- En-dashes, em-dashes, ellipses, opening punctuation (Spanish `¿`, `¡`) —
  always outside spans.
- Capitalization at sentence start — store the lowercased form in the TOML key,
  but render the original casing in the visible text. The `.w` wrapper preserves
  text content.

---

## Part 5 — Worked example

A complete miniature example for **French → English**, A1 level. Shows the
shape of all three pipeline outputs at minimum size.

### `stories/01-le-chat/story.md`

```markdown
---
slug: 01-le-chat
title: "Le chat sur le toit"
date: 2026-05-11
words: 18
protagonist: "le chat"
setting: "un toit de Lyon, en hiver"
tone: "quiet, observational, slightly absurd"
new_keywords:
  - toit
  - regarde
---

Le chat dort sur le toit. La nuit est calme.

Il rêve d'une lune en fromage. Personne ne le regarde.
```

### `stories/01-le-chat/enrichment.toml`

```toml
[meta]
slug = "01-le-chat"
title = "Le chat sur le toit"
target_lang = "fr"
native_lang = "en"

[words."le"]
tr = "the (m.)"
pos = "det"
lemma = "le"

[words."chat"]
tr = "cat"
pos = "noun"
lemma = "le chat"

[words."dort"]
tr = "(he) sleeps"
pos = "verb"
lemma = "dormir"
grammar = """
**dort** — *present*, 3sg, from **dormir** ("to sleep").

Irregular verb, but the 3sg ending is regular.

`dor·[[t]]`
"""

[words."sur"]
tr = "on, on top of"
pos = "prep"
lemma = "sur"

[words."toit"]
tr = "roof"
pos = "noun"
lemma = "le toit"

[words."la"]
tr = "the (f.)"
pos = "det"
lemma = "la"

[words."nuit"]
tr = "night"
pos = "noun"
lemma = "la nuit"

[words."est"]
tr = "is"
pos = "verb"
lemma = "être"
grammar = """
**est** — *present*, 3sg, from **être** ("to be").

Highly irregular — memorize the full paradigm: `suis · es · est · sommes · êtes · sont`.
"""

[words."calme"]
tr = "calm, quiet"
pos = "adj"
lemma = "calme"

[words."il"]
tr = "he, it"
pos = "pron"
lemma = "il"

[words."rêve"]
tr = "(he) dreams"
pos = "verb"
lemma = "rêver"
grammar = """
**rêve** — *present*, 3sg, from **rêver** ("to dream").

Regular `-er` verb: 3sg → `-e`.

`rêv·[[er]]` → `rêv·[[e]]`
"""

[words."d'"]
tr = "of"
pos = "prep"
lemma = "de"
grammar = """
**d'** — elided form of **de** before a vowel.

`de` + `une` → **d'une**.
"""

[words."une"]
tr = "a (f.)"
pos = "det"
lemma = "une"

[words."lune"]
tr = "moon"
pos = "noun"
lemma = "la lune"

[words."en"]
tr = "in, made of"
pos = "prep"
lemma = "en"

[words."fromage"]
tr = "cheese"
pos = "noun"
lemma = "le fromage"

[words."personne"]
tr = "nobody"
pos = "pron"
lemma = "personne"
grammar = """
**personne** is the negative pronoun "nobody". Combined with **ne**: `ne ... personne`.

Without `ne`, *personne* as a noun means "person" — context disambiguates.
"""

[words."ne"]
tr = "not (part of negation)"
pos = "adv"
lemma = "ne"

[words."regarde"]
tr = "(he) watches, looks at"
pos = "verb"
lemma = "regarder"

[[sentences]]
fr = "Le chat dort sur le toit."
tr = "The cat sleeps on the roof."

[[sentences]]
fr = "La nuit est calme."
tr = "The night is quiet."

[[sentences]]
fr = "Il rêve d'une lune en fromage."
tr = "He dreams of a moon made of cheese."

[[sentences]]
fr = "Personne ne le regarde."
tr = "Nobody watches him."
```

### `stories/01-le-chat/index.html` — wrapping fragment

The key body fragment showing one wrapped paragraph. Full design would also
include `<head>` with Google Fonts, inline `<style>` with the bespoke palette
(say, slate blue + parchment + a single warm amber), a hand-drawn SVG of a cat
silhouette on a tile roof, the popup mount, and inline `<script>` with
`mdToHtml()` and the popup handlers.

```html
<p>
  <span class="w" data-tr="the (m.)" data-pos="det" data-lemma="le">Le</span>
  <span class="w" data-tr="cat" data-pos="noun" data-lemma="le chat">chat</span>
  <span class="w"
        data-tr="(he) sleeps"
        data-pos="verb"
        data-lemma="dormir"
        data-grammar="**dort** — *present*, 3sg, from **dormir** (&quot;to sleep&quot;).&#10;&#10;Irregular verb, but the 3sg ending is regular.&#10;&#10;`dor·[[t]]`">dort</span>
  <span class="w" data-tr="on, on top of" data-pos="prep" data-lemma="sur">sur</span>
  <span class="w" data-tr="the (m.)" data-pos="det" data-lemma="le">le</span>
  <span class="w" data-tr="roof" data-pos="noun" data-lemma="le toit">toit</span>.
</p>
```

Note:

- The period is outside any `.w` span — it gets a `.s` span instead, paired
  with the corresponding `[[sentences]].tr` translation.
- `data-grammar` newlines are encoded as `&#10;`, quotes as `&quot;`.
- `Le` (capitalized) keeps its original casing; the TOML key was `le`.
- Each `.w` carries everything the popup needs — the JS reads `dataset`
  properties and renders the popup from those alone. No external state.
- **The designer doesn't write this fragment by hand.** They write the page
  with `<p>Le chat dort sur le toit.</p>` inside `<article data-story-body>`,
  then run the helper script which produces the wrapped output above. The
  fragment is shown here so the reader understands what the script emits.

The full single-file render is: the designer's `<head>` (including their
inline `<style>` with the bespoke palette and any Google Fonts links), their
`<body>` with the back-link, hero SVG, and the `[data-story-body]` element
holding the paragraphs — plus the script-injected `<style data-popup-invariants>`
at end of `<head>` and `<script data-popup>` before `</body>`.

---

## Part 6 — Anti-checklist

Things the agent must NOT do, even if it seems helpful.

### Architecture

- ❌ Shared base HTML template across stories.
- ❌ External JS libraries or frameworks (jQuery, Tailwind, Bootstrap, React,
  Vue, …).
- ❌ Analytics, trackers, GTM, Mixpanel, anything that phones home.
- ❌ External CDNs other than Google Fonts (and only when the user opted in).
- ❌ Raster image files. Illustrations are inline SVG only. Motion comes from inline animated SVG (CSS keyframes or SMIL) or pure CSS animations — never raster GIFs, never as data URIs.
- ❌ Auto-generated grammar engines that synthesize from POS + lemma. Curate
  data, parse with `mdToHtml()`, stay small.
- ❌ Reusing `data-grammar` text across stories by referencing a shared
  glossary — each story carries its own grammar inline, so the renderer JS
  stays trivial.

### Content

- ❌ Native-language glosses inline in the story body. Translations only in
  popups (and the optional sentence-mode).
- ❌ Inline learning sidebars, vocabulary lists, "key words to study" boxes.
  The popup IS the vocabulary affordance.
- ❌ Moralizing endings, romance, or any anti-vibe the user listed in
  `profile.md`.
- ❌ Cliffhangers. Self-contained means self-contained.
- ❌ Lorem ipsum or placeholder text anywhere.
- ❌ Emoji as primary illustration. (Unicode dingbats as small typographic
  accents are OK if the design calls for it.)
- ❌ Generic "story page" output — a render that could be lifted onto any
  other story by swapping color and font. Each render commits to a specific
  in-universe artifact (see [Found-document framing](#found-document-framing)).
- ❌ Reusing a format archetype within the previous 3 stories
  (see [Format archetypes — variety rule](#format-archetypes--variety-rule)).
  After three stories away, a returning archetype must be a visibly fresh
  interpretation, never a re-skin.
- ❌ Profession-as-decorative-veneer — a few thematic SVGs glued onto an
  otherwise-generic centered serif column. The artifact framing must shape
  page chrome, metadata, layout, and motion, not just illustration.
- ❌ Defaulting to a narrow horizontally centered single-column body. That
  layout is allowed only when the story explicitly benefits from minimalism
  (intimate monologue, brief confessional, museum-placard quiet); otherwise
  the page must use the editorial / cinematic / spatial toolkit in
  [Layout philosophy](#layout-philosophy--editorial-cinematic-spatial).
- ❌ Decorative pass that pulls the reader's eye before the prose does.
  Ambient gradients, framing chrome, marginalia, and split-pane visuals
  stay quieter than the body text — pull quotes are the only allowed
  exception, and only because they *are* story text.

### Implementation

- ❌ Editing legacy story HTML files when refreshing the project index. Leave
  them alone unless the user explicitly asks for a migration.
- ❌ Renaming slugs after publication.
- ❌ Auto-fixing the user's existing custom files without asking.
- ❌ Bypassing `pointer-events:none` on the hidden popup.
- ❌ Bridge taller than ~12px.
- ❌ Hand-tokenizing the body. Wrap your `<p>` tags inside an element with
  `data-story-body` and let the helper script do it — it handles escaping,
  sentence pairing, phrase matching, and the popup invariants correctly.
- ❌ Hand-editing the auto-managed `<style data-popup-invariants>` or
  `<script data-popup>` blocks. Style popups via your own CSS (declared above
  the auto-managed block in source order); the script replaces these blocks on
  every run.
- ❌ Skipping the `data-story-body` attribute when designing a new page. The
  script needs it to find the body region; without it, it errors out rather
  than guessing.
- ❌ Unquoted non-ASCII TOML keys (`[words.señal]`). TOML bare keys are
  ASCII-only; quote the form (`[words."señal"]`). See
  [Part 3.2 — TOML key quoting](#toml-key-quoting).
- ❌ Fast, loud, or attention-grabbing motion. Ambient layers must be slow
  (multi-second cycles), low-opacity, and behind text. The body text itself
  never moves once loaded.
- ❌ JS-driven motion that ignores `prefers-reduced-motion`. Gate JS
  animations on `matchMedia` or use CSS `animation` / `transition` so the
  auto-injected guard applies.
- ❌ Body text below 18 px or contrast ratio 4.5:1, even inside framed
  panels, marginalia, or split panes. Real font sizes everywhere the
  reader actually reads.
- ❌ Layouts that break on narrow viewports. Multi-column, split-screen,
  and floating regions must collapse to a clean single column on mobile,
  with no horizontal scroll and no overlapping text.
- ❌ Replacing semantic `<p>` / `<h1>` / `<blockquote>` / `<aside>` with
  `<div>` salad to hit a visual. The tokenizer and assistive tech both
  depend on real structure inside `<article data-story-body>`.

### Tone

- ❌ Treating the user as a beginner who needs encouragement. They're a
  language learner, not a child.
- ❌ Adding "Did you know?" trivia callouts.
- ❌ Adding "Try saying this aloud!" prompts.
- ❌ Generic AI illustrations of "books and globes and chat bubbles".

---

## Part 7 — Iteration loop

After each story, the user reads it and gives feedback. The agent's job is to
refine `profile.md` based on that feedback so subsequent stories drift toward
what the user actually enjoys. Expect the first few stories to be calibration
shots — tone, level, and shared-universe feel typically need 2–3 iterations to
settle.

### What to write back into `profile.md`

- **Topic adjustments** — "they liked the navigator-in-deep-space angle, less
  the politics" → narrow the topic line, add a clarifying anti-vibe line.
- **Tone refinements** — "they said story 2 felt too dark" → adjust the tone
  spectrum entry.
- **Pacing** — "they said 270 words felt right, 350 was too long" → adjust
  words-per-text.
- **Vocabulary** — "they said 30% of words were new, too much" → adjust the
  new-vocab ratio.
- **Reference works** — "they mentioned they also liked Le Guin" → add to
  reference list with the specific resonant quality.

Each iteration of `profile.md` should be timestamped in a small changelog at
the bottom so the user can see drift.

### What NOT to write into `profile.md`

- Specific story premises ("write more stories about Marco"). If the user
  explicitly wants serialization, capture those in a separate companion file
  (the name is up to the user — `series.md`, `arcs.md`, whatever) so
  `profile.md` stays a stable taste record, not a serial bible.
- Ephemeral session moods ("user seemed tired today").
- Anything that contradicts what's already there — surface the contradiction
  to the user, let them resolve it.

### When to recalibrate from scratch

If the user's level rises (A1 → A2 → B1), redo Section B of the interview but
leave the taste profile alone. If their taste materially shifts (they got tired
of sci-fi and want food writing), redo Section C but keep level intact. Don't
nuke the whole profile unless asked.

---

## Closing note to the AI agent reading this

Generate the project on disk. Build the three skills (story, enrich, render),
including the render helper script. Run the interview, write `profile.md`.
Then offer to generate the first story — let the user steer the topic. Step
back and let them read.

The render flow is: design first (you write `index.html` freely with
`data-story-body` on the body wrapper), then run the helper script to apply
enrichment. Don't hand-tokenize the body. Don't hand-edit the auto-managed
popup CSS / JS blocks. Style popups via your own CSS.

If the user gives feedback like "popups are flickering", check the invariants
in Part 4 first — those are the high-probability bugs and they should be
satisfied by the script's auto-injected `<style data-popup-invariants>` block;
if not, the script's invariants block is wrong. If the user gives feedback
like "the stories feel generic", revisit the taste-profile section of the
interview and push for more specificity. If the user gives feedback like "the
design feels generic", you're likely defaulting to AI-template aesthetics —
invoke the `frontend-design` skill (or its equivalent in your agent) and
commit to a bolder direction.

If the user asks you to add a feature this blueprint forbids (a shared base
template, an auto-grammar engine, external JS libraries), explain the tradeoff
and let them override the blueprint consciously, not by accident.

Expect the first two or three stories to feel slightly off — they're
calibration shots. Tone, length, and shared-universe density usually settle by
the fourth. Capture each iteration's adjustments back into `profile.md` (see
[Part 7](#part-7--iteration-loop)) so the drift is visible and the user can
audit it later.
