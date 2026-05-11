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
| **Render** | The hand-designed `index.html` page produced for one story. Each story gets its own design — no shared base template. |
| **Hover popup** | The UI affordance shown when the user hovers (or taps) a wrapped word in the rendered HTML. Contains translation, POS, lemma, dictionary link, optional grammar block. |
| **Grammar mini-language** | A tiny markdown-ish syntax (`**form**`, `*term*`, `` `code` ``, `[[suffix]]`) used inside `enrichment.toml`'s `grammar` field, parsed at popup-show time by an inline JS function `mdToHtml()`. |
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
│       └── render/SKILL.md      ← writes per-story index.html + refreshes project index
└── stories/
    └── NN-slug/             ← one folder per story
        ├── story.md         ← Spanish/French/Japanese/... body + frontmatter
        ├── enrichment.toml  ← per-word, per-sentence, per-phrase data
        └── index.html       ← bespoke render
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

**Inputs**:
- `stories/NN-slug/story.md`
- `stories/NN-slug/enrichment.toml`
- `profile.md`
- The existing project `index.html` (to preserve other entries when refreshing)

**Output**:
- `stories/NN-slug/index.html` (new)
- `index.html` (updated)

This skill has two contracts: **functional** (the popup system must work the
same across all stories) and **design** (every story must look different).

#### Functional contract — same for every story

##### Word wrapping

Every meaningful word in the body is wrapped:

```html
<span class="w"
      data-tr="brief native-language gloss"
      data-pos="noun|verb|adj|adv|...|proper"
      data-lemma="dictionary form"
      data-grammar="**form** — *category*&#10;&#10;...">word</span>
```

- `data-grammar` is OPTIONAL and only present if the TOML had a `grammar` field.
- Newlines inside `data-grammar` are encoded as `&#10;` (HTML numeric entity).
  Double quotes inside any attribute are encoded as `&quot;`.
- Punctuation, numerals, and proper nouns stay **outside** `.w` spans
  (unless flagged as a `[[phrases]]` entry, in which case the phrase becomes one
  span).

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
- **Layout / composition**: vary across stories. Single-column centered,
  asymmetric, side-margin metadata, faux-interface frames (log-entry consoles,
  letter paper, hazard documents). The composition is part of the storytelling.
- **Reading width**: ~640–720px max for body text. Never less than 18px font
  size. Line-height 1.7–1.9 for low-CEFR readers, 1.6–1.7 for higher.

The page should feel like the interior of a book imagined for **this specific
story**, not a generic learner-app template.

#### Project-index refresh

After writing the story HTML, the render skill **also** updates the project
`index.html`:

- Read it.
- Locate the `<!-- STORIES:START -->` / `<!-- STORIES:END -->` markers. If
  either is missing, **stop and tell the user** rather than guessing — the
  markers are the contract; without them, "the entries list" isn't safely
  defined.
- Insert a new `<a class="entry" href="stories/NN-slug/index.html">…</a>` block
  immediately before `<!-- STORIES:END -->`.
- Update the entry-count badge (e.g. `03 entradas` → `04 entradas`).
- Preserve every existing entry between the markers verbatim — including the
  user's hand-edits (custom hrefs, reordered rows, manual metadata tweaks).
  The render skill only **adds** rows; it doesn't normalize or "fix" what the
  user touched.
- Preserve everything outside the markers byte-for-byte — header, footer,
  styles, scripts, design. This file is a stable artifact, not a per-render
  bespoke surface.

#### Back-link

Each story page has a small back-link to the project index. Path is
`../../index.html` from `stories/NN-slug/index.html`.

#### Workflow

1. Read `story.md`, `enrichment.toml`, `profile.md`.
2. Decide design direction (palette, fonts, illustration motif, layout).
   Briefly — internal, not shown to user.
3. Tokenize the body. For each word, look up `[words."form"]` in the TOML,
   build the `<span class="w" ...>` wrapper. Use `[[phrases]]` for multi-word
   units (longest-match-wins).
4. Build the full single-file HTML:
   - `<!doctype html>`, `<html lang="<target>">`, `<meta charset="utf-8">`,
     `<meta name="viewport">`
   - `<title>` from frontmatter
   - Google Fonts `<link rel="preconnect">` + stylesheet (only if
     `External fonts: yes` in `profile.md`; omit entirely otherwise)
   - Inline `<style>` with all design CSS
   - `<body>` containing the back-link, optional metadata strip, hero SVG,
     story body
   - One `<div class="pop"></div>` mount point
   - Inline `<script>` containing `mdToHtml()` plus popup behavior (~50–100
     lines)
5. Write to `stories/NN-slug/index.html`.
6. Update the project `index.html` per above.
7. Report: file path, one-sentence thematic summary (palette, font pair, motif),
   and a hint command to open in browser.

---

## Part 4 — Critical invariants

These are bugs already paid for in the baseline implementation. Don't repeat them.

### Invariant 1 — Popup pointer-events when hidden

The hover popup `<div class="pop">` is mounted once at document level and
positioned absolutely over the body. When **hidden**, it must NOT block pointer
events on the words underneath, or it will silently swallow `mouseenter` on
any word it happens to be sitting over from its previous show.

```css
.pop {
  pointer-events: none;   /* default: invisible AND non-blocking */
  opacity: 0;
  visibility: hidden;
}
.pop.show {
  pointer-events: auto;   /* only intercept events while shown */
  opacity: 1;
  visibility: visible;
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
  left: 0;
  right: 0;
  height: 12px;
}
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

### Invariant 8 — Word-boundary edge cases

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

- The period is outside any span.
- `data-grammar` newlines are encoded as `&#10;`, quotes as `&quot;`.
- `Le` (capitalized) keeps its original casing; the TOML key was `le`.
- Each `.w` carries everything the popup needs — the JS reads `dataset`
  properties and renders the popup from those alone. No external state.

This fragment plus the inline `<style>`, `<script>`, hero SVG, and popup mount
constitute the complete single-file render.

---

## Part 6 — Anti-checklist

Things the agent must NOT do, even if it seems helpful.

### Architecture

- ❌ Shared base HTML template across stories.
- ❌ External JS libraries or frameworks (jQuery, Tailwind, Bootstrap, React,
  Vue, …).
- ❌ Analytics, trackers, GTM, Mixpanel, anything that phones home.
- ❌ External CDNs other than Google Fonts (and only when the user opted in).
- ❌ Raster image files. Illustrations are inline SVG only.
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

### Implementation

- ❌ Editing legacy story HTML files when refreshing the project index. Leave
  them alone unless the user explicitly asks for a migration.
- ❌ Renaming slugs after publication.
- ❌ Auto-fixing the user's existing custom files without asking.
- ❌ Bypassing `pointer-events:none` on the hidden popup.
- ❌ Bridge taller than ~12px.

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

Generate the project on disk. Build the three skills. Run the interview, write
`profile.md`. Then offer to generate the first story — let the user steer the
topic. Step back and let them read.

If the user gives feedback like "popups are flickering", check the invariants
in Part 4 first — those are the high-probability bugs. If the user gives
feedback like "the stories feel generic", revisit the taste-profile section of
the interview and push for more specificity.

If the user asks you to add a feature this blueprint forbids (a shared base
template, an auto-grammar engine, external JS libraries), explain the tradeoff
and let them override the blueprint consciously, not by accident.

Expect the first two or three stories to feel slightly off — they're
calibration shots. Tone, length, and shared-universe density usually settle by
the fourth. Capture each iteration's adjustments back into `profile.md` (see
[Part 7](#part-7--iteration-loop)) so the drift is visible and the user can
audit it later.
