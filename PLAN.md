# Exam Scorer — Build Plan: Reading, Listening, Use of English

Written from the two front-end files only (`index.html`, `admin.html`). The Cloudflare
Worker (`https://exam-scorer.kayuso2011.workers.dev`) was NOT visible when this was
written. Everything marked **[verify in Worker]** is an assumption to check against the
real Worker code before building on it.

Execute this on the PC, with the Worker repo open next to this one.

---

## 1. Current state

| Skill | Student side (`index.html`) | Teacher side (`admin.html`) |
|---|---|---|
| Writing | Done, all exams | Roster, quotas, history |
| Speaking | Done (record, transcribe, paywall) | Grant/revoke access |
| Reading | Built for `ege`, `oge` only. Formats: `matching`, `tfns`, multiple choice. Free, objectively marked, no model call | Generate, review, approve, retire |
| Listening | **No student tab or pane.** | Bank can already generate/review `listening` for ege/oge |
| Use of English | **Nothing anywhere** | **Nothing anywhere** |

Exams in the app today (`EXAM_GROUPS`, `index.html` ~line 844): `ielts, toefl, det`,
`fce, cae, cpe, pet, ket`, `oge, ege`. **SAT is not an exam key yet.** The only SAT trace
is a comment in the Reading renderer about "discrete-stimulus" items (each question
carries its own `stimulus`), so the Worker may already have part of this **[verify in
Worker]**.

### How the existing Reading flow works (the contract to extend)

Endpoints the front end calls today:

- `GET /exams` -> `{ exams: {key: label}, sectionExams: [keys with a section bank] }`
- `GET /section/available?exam=&skill=reading` ->
  `{ section: {label, maxPrimary, tasks:[from,to], spec...}, groups: [{taskFrom, taskTo, itemCount, available, ...}] }`
- `POST /section/start { studentCode, examKey, skill, taskFrom }` ->
  `{ bankId, format, passage, items }`
- `POST /section/submit { studentCode, bankId, answers: [int|null] }` ->
  `{ correct, total, percent, perItem: [{correct, chosen, expected}], items, format, points?, pointsUnavailableReason?, bandUnavailableReason? }`
- Admin (header `X-Admin-Key`): `GET /admin/bank/groups?exam=&skill=`,
  `POST /admin/bank/generate { examKey, skill, taskFrom }` (422 = unusable set, retry),
  `GET /admin/bank?status=draft|approved|retired`, `POST /admin/bank/review { id, action: approve|retire, reason }`

Item shapes:

- MCQ / TFNS: `items: [{ q, options[], stimulus?, answer (index, admin only), why, whyNot[] }]`
- Matching: `items: { targets[], options[], answer[], why[] }`

Answers are always integers (option indexes) or `null`. That is the main thing that
breaks for Use of English (typed answers) — see Phase 3.

Quality gate already in place: Worker structure check -> second model solves the set
cold -> teacher approves in admin. Keep this for every new format.

---

## 2. Target matrix (what each exam gets)

"Reading" and "Listening" below mean the objectively-marked bank sections. Writing and
Speaking already exist for every exam.

| Exam | Reading | Listening | Use of English | Notes |
|---|---|---|---|---|
| ЕГЭ / ОГЭ | done | Phase 1 | no | FIPI spec, Listening tasks 1-9 (ЕГЭ) / 1-8 (ОГЭ) — check against current demo |
| IELTS | Phase 4 | Phase 4 | no | Academic Reading 3 passages/40 Qs; Listening 4 parts/40 Qs. Band conversion is published — use it |
| TOEFL iBT | Phase 4 | Phase 4 | no | Current format (2026 revision). Confirm task types in ETS guide before writing the spec |
| DET | Phase 5 | Phase 5 | no | Adaptive, integrated question types (Read & Select, Fill the Blanks, Listen & Type, Read & Complete, Interactive Reading/Listening). Does not fit "passage + N questions" — see Phase 5 |
| SAT | Phase 5 | none | no | Reading & Writing only, no Listening. Digital SAT: short single-stimulus items. Needs a new exam key |
| FCE / CAE / CPE | Phase 4 | Phase 4 | **Phase 3** | "Reading and Use of English" paper. Check which format version you teach (Cambridge has been revising FCE/CAE) |
| PET / KET | Phase 4 | Phase 4 | no | PET/KET are "Reading and Writing" combined, no Use of English |

Use of English is **Cambridge FCE, CAE, CPE only**:

- Part 1 multiple-choice cloze (MCQ, reuses existing format)
- Part 2 open cloze (typed single word)
- Part 3 word formation (typed word, derived from a prompt word)
- Part 4 key word transformations (typed 2-5 words, using a given key word)
- CAE/CPE: Part 4/5 variations — confirm per level in the Cambridge handbook

---

## 3. Phases (build in this order)

### Phase 0 — Prep (before writing code)

1. Open the Worker repo beside this one. Read: the route table, the section spec module
   (where ege/oge tasks, `maxPrimary`, groups and `supported` flags are defined), the
   bank generate/solve/gate code, and the D1/KV schema for bank entries.
2. Confirm which of the contract points above are real. Write the differences at the
   bottom of this file under "Verified contract".
3. Decide the data model change once, up front (Phase 3 forces it): answers can be
   `int | null | string`. Pick one:
   - **Recommended:** keep `answers` an array but allow strings for `gap` items; marking
     branches on `items[i].type`.
   - Alternative: a separate `/section/submit` payload shape per format. More code, no
     benefit.
4. Get the official specs/demos for every exam you will add (FIPI demo, IELTS, ETS,
   Duolingo technical manual, College Board SAT spec, Cambridge handbooks). Section
   specs must come from these, never from the model's memory — this app's whole pitch
   is "graded against real criteria".

### Phase 1 — Listening (student side) for ЕГЭ/ОГЭ  *(teacher side already works)*

Worker:

- `GET /section/available?skill=listening` and `POST /section/start` must serve
  `listening` bank entries. The Reading code path probably already does if `skill` is
  generic **[verify in Worker]**.
- Return an `audio` descriptor on start. Pick one approach:
  - **A (simplest, matches Speaking):** return the transcript as `script` (array of
    turns/lines, with optional speaker/voice) and let the browser speak it. The app
    already has browser TTS and an IndexedDB clip cache (`speakLine`, `clipGet/Put`).
    Cost: voice quality varies by device.
  - **B (better quality):** Worker synthesizes audio once at approval time, stores in R2,
    returns a URL. Needs R2 binding and a TTS provider. Do this later if A isn't good
    enough.
- **Do not return the transcript until after submit.** Return it in the submit response
  instead (so students can review with the script). For approach A the script has to go
  to the client to be spoken — accept that, but never render it, same as the existing
  voiced Speaking tasks.
- Play rules from the spec (ЕГЭ/ОГЭ play each recording twice, with a fixed pause):
  `playsAllowed`, `pauseSeconds` in the start response; enforce client-side, record plays
  server-side only if you need anti-cheat (you don't).

Front end (`index.html`):

1. Add a `Listening` pill: `data-skill="listening"` next to Reading (line ~419).
2. Add `#listeningPane` modelled on `#readingPane` (line ~540): task picker, Start,
   a player area, then the questions column and Check button.
3. Generalise `loadReadingTasks / startReading / renderReadingQuestions / submitReading /
   renderReadingResult` into `loadSectionTasks(skill)`, etc. They hard-code
   `skill: "reading"` and `readingQs`. Reuse, do not copy-paste.
4. `onSkillChange()` (line ~1036): add the listening branch; hide the writing and
   speaking panes, show the new one. Add the same stranded-pane guard that
   `syncReadingPill` has.
5. Player: a "Play" button that speaks `script` via `speakLine`, counts plays, disables
   after `playsAllowed`, shows the pause countdown between plays, and cancels speech if
   the student switches skill (see how Speaking does this around line 1061).
6. Add `sectionExams` per skill (e.g. `{reading: [...], listening: [...]}`) from `/exams`,
   because Listening and Reading will not cover the same exams (SAT has no Listening).
   `syncReadingPill` becomes `syncSectionPills`. Disabled-pill tooltip text is currently
   Reading-specific Russian.
7. Result screen: add a transcript reveal after submit.
8. Update pricing and about copy (line ~628, ~1857): "Чтение — ОГЭ и ЕГЭ" becomes a list
   per skill.

Done when: an approved ege listening entry can be taken end to end, plays at most the
allowed number of times, and marks correctly.

### Phase 2 — Admin generalisation

`admin.html`:

- `#bankExam` (line ~182) lists only ege/oge. Populate it from `/exams` `sectionExams`
  instead of hard-coding.
- `#bankSkill` gets `use` (Use of English) once Phase 3 lands.
- `BANK_SKILL_LABEL` (line ~586): add `use`.
- `renderBankItems` (line ~810) handles matching and MCQ only. Add a renderer per new
  format (`gap`, `transform`) showing the accepted answers list. Everything with an
  answer key must show its key so the teacher can verify it.
- Gate display: for typed-answer formats the "checker agreed N/M" tally needs the
  checker to produce a string answer and compare with the accepted list — decide what
  "agreed" means there.

### Phase 3 — Use of English (FCE, CAE, CPE)

New formats. Item shape proposal:

```jsonc
// Part 1 — multiple-choice cloze: existing MCQ, but gaps are inline
{ "type": "mcq-cloze", "gap": 1, "options": ["...", "...", "...", "..."], "answer": 2, "why": "..." }

// Part 2 — open cloze
{ "type": "gap", "gap": 9, "accepted": ["of", "off"], "why": "..." }

// Part 3 — word formation
{ "type": "gap", "gap": 17, "prompt": "ORGANISE", "accepted": ["disorganised", "disorganized"], "why": "..." }

// Part 4 — key word transformation
{ "type": "transform", "sentence": "...", "keyword": "BLAME", "lead": "He ...", "tail": "...", "accepted": ["wasn't to blame for"], "why": "..." }
```

Worker:

- Section specs for `fce/cae/cpe` skill `use`, with Part -> task groups and max marks
  from the Cambridge handbook. Part 4 is scored 0-2 per item in the real exam (1 per
  half). Do not collapse to 0/1 without saying so in the UI.
- Generation prompts per part. The generator is the weak link: word formation and key
  word transformations have multiple valid answers. Require the generator to output ALL
  acceptable answers, and have the solver model propose answers independently; flag
  mismatches for the teacher exactly as now.
- Marking: normalise (trim, lowercase, collapse spaces, strip trailing punctuation),
  compare against `accepted`. For transforms, compare the gap text only, enforce word
  count (2-5 words, contractions count as 2), and require the key word to be present
  unchanged. Spelling must be exact (as in the real exam), but accept UK and US spelling
  where listed in `accepted`.
- Submit endpoint accepts string answers (Phase 0 decision) and returns for each item:
  `expected` (first accepted), `acceptedAll`, `correct`, plus the student's text.

Front end:

- New renderer branch for `gap` (inline text input inside the passage, numbered; or a
  list below for narrow screens) and for `transform` (original sentence, key word in
  bold, lead-in, an input, tail). Passage text is plain `textContent` today — gaps need
  a token such as `{{9}}` in the passage that the renderer swaps for `<input>`. Build
  with DOM nodes, not `innerHTML`.
- `readingAnswerCount` assumes integers (`Number.isInteger`). Replace with a type-aware
  "is answered".
- Result: show student's text and the accepted answers side by side; whyNot is not
  applicable, `why` is.
- A `Use of English` pill, enabled only for fce/cae/cpe.

Done when: one full Part 1-4 set per level can be generated, reviewed, approved and
taken, and an alternative valid answer (e.g. `organized`) is marked right.

### Phase 4 — Reading + Listening for IELTS, TOEFL, Cambridge (all five levels)

Per exam: write the section spec, then the generation prompts, then verify with the
solver gate, then teacher approval. New formats needed beyond Phase 3:

- IELTS Reading: True/False/Not Given (exists as `tfns`), Yes/No/Not Given (same shape,
  different labels), matching headings (exists), matching information, sentence
  completion / summary completion (typed, reuse `gap`), multiple choice (exists).
- IELTS Listening: form/note completion (typed `gap`), MCQ, map labelling (needs an
  image — defer or skip), matching.
- TOEFL: MCQ plus the newer task types; read the current ETS guide first. Listening
  stimuli are conversations/lectures: `script` has multiple speakers.
- Cambridge Reading (non-UoE): gapped text (MCQ over removed sentences — new
  `sentence-removal` type, or reuse matching), multiple matching, MCQ. Listening: MCQ,
  sentence completion (`gap`), multiple matching.

Because these reuse Phase 1 + 3 formats, this phase is mostly Worker specs and prompts,
plus very small front-end changes (labels per format).

Scoring: report raw score always; report a band/scale only where the conversion table is
published (IELTS bands are; Cambridge scale scores need per-test conversion — show raw
only). Reuse the existing `pointsUnavailableReason` pattern.

### Phase 5 — SAT and DET

These do not fit the existing pattern; scope them separately.

SAT:

- Add exam key `sat` to `EXAM_GROUPS`, `FALLBACK_LABELS`, the Worker exam table, and the
  Writing/Speaking behaviour (SAT has neither as a scored section — make sure the app
  does not offer Writing/Speaking tabs for it, or decide to offer essay-style practice
  anyway).
- Reading and Writing section only: discrete items, each with its own `stimulus`. The
  renderer already supports `stimulus` per item and hides the left column when `passage`
  is empty. The comment says the Worker may already emit this **[verify in Worker]**.
- Digital SAT scoring is adaptive and scaled; do not invent a score. Raw correct only.

DET:

- Question types are many small interactive items, not passages. Prioritise the ones
  that are objectively markable: Read and Select (pick real words), Fill in the Blanks
  (typed letters), Listen and Type (typed sentence, tolerant marking), Read and Complete
  (letters missing in words). Interactive Reading/Listening and the production tasks are
  outside the scope of this bank (production already goes through Writing/Speaking).
- New item types: `word-select` (checkbox list of real/fake words), `letters-gap`
  (missing letters, typed), `dictation` (typed sentence compared with tolerance).
- Do not present a 10-160 DET score. Raw only, with a note.

### Phase 6 — Cross-cutting

- Progress tab (`#progressSection`) tracks writing/speaking; decide whether section
  results (raw %, per skill, per exam) get recorded server-side per student and shown
  to the teacher in the admin roster. Currently Reading results are not stored against
  the student in the visible code (the submit call sends `studentCode` though — check
  what the Worker does with it **[verify in Worker]**).
- Rate limiting and abuse: section endpoints are free and unauthenticated aside from
  the student code. Generation is admin-only, so cost is bounded. Confirm.
- Russian-language UI strings for new panes (the student UI is Russian; error strings
  from the Worker are English — the Reading code already handles that case).
- Pricing/copy and legal text (`index.html` lines ~624-670, 760-840) mention which
  exams have Reading; update when a new exam/skill goes live. These are legal pages for
  a Russian merchant — have the text reviewed rather than letting a model rewrite it.

---

## 4. Suggested order and size

| Step | Work | Size |
|---|---|---|
| 0 | Read Worker, record verified contract here | S |
| 1 | Listening student side, ege/oge | M |
| 2 | Admin: dynamic exam list, new skill labels, new renderers | S |
| 3 | Use of English (gap/transform formats + FCE/CAE/CPE specs) | L |
| 4 | IELTS, TOEFL, PET/KET/FCE/CAE/CPE Reading and Listening | L (mostly spec + prompts) |
| 5a | SAT | M |
| 5b | DET | L |
| 6 | Progress storage, copy, legal | S–M |

Ship after each step; each is independently useful. Do not start Phase 4 until Phase 3's
answer model is settled, because Phase 4 reuses it.

## 5. Risks and decisions to make

1. **Spec accuracy.** Wrong task counts or timings would make this a bad practice tool.
   Source every spec from the official document and put the citation in the spec module
   (the admin spec note already prints "Structure from the ФИПИ спецификация").
2. **Generated key quality.** Typed-answer items have the highest wrong-key risk. Keep
   teacher approval mandatory and never auto-approve even when the checker agrees.
3. **TTS quality (Listening).** Start with browser TTS; judge on real devices before
   investing in R2 + a TTS provider.
4. **Cambridge format versions.** Confirm which FCE/CAE format you want to follow
   before writing the specs.
5. **Exam coverage vs. effort.** DET and SAT cost more than the rest combined.
   Reasonable to ship Phases 0-4 first.

## 6. Verified contract (fill in on the PC)

_(Record here the real routes, schemas, spec module names and any difference from
section 1.)_
