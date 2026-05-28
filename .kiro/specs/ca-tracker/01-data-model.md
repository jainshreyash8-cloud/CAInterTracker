# CA Inter Tracker — Data Model Spec (the "spine")

> **Purpose:** Lock down *what* we are tracking and *how the pieces relate*, before any tab gets built. Every tab in the app is a different view or entry point into the entities defined here. If a tab needs a piece of data that isn't in this document, **this document is wrong** and should be updated first.
>
> **Status:** DRAFT — review and mark up. Open questions are flagged with `❓` and listed at the bottom.

---

## 1. Design principles

These are the rules that drove the entire model. If a design decision conflicts with one of these, the decision is wrong.

1. **Time is the schema, not a feature.** Every activity a student does is, at its root, a single `TimeEntry`. The Timeline tab is just a visualization of this collection. The Dashboard is just aggregates over it.
2. **Time is not the same as progress.** Sitting with a lecture for 60 minutes and consuming 40 minutes of lecture content are two different numbers. We store both.
3. **Three entry contexts, never collapsed into one form.** *Schedule / Add* (before doing it), *Session* (while doing it), *Quick Complete* (after the fact). Each one asks for only the fields it can know at that moment.
4. **Granularity follows real-world atoms.** A chapter has topics. A first reading happens in chunks. A chunk has multiple revisions. Every level exists because something downstream needs it.
5. **No phases, only goals.** Phases assume a clean linear path. Real CA prep overlaps. Goals run in parallel with their own deadlines.
6. **Pomodoro is a session option, not a separate activity type.** Activity = *what you're doing*. Pomodoro = *how you're doing it*.
7. **No stacked countdowns.** A single "reference time" line per session is enough.

---

## 2. Core entities

### 2.1 Subject

Already defined in code. 8 subjects, mapped onto 6 ICAI papers:

| Subject id | Name | Group | Paper |
|---|---|---|---|
| `aa` | Advanced Accounting | 1 | P1 |
| `law` | Corporate & Other Laws | 1 | P2 |
| `it` | Income Tax | 1 | P3 |
| `gst` | GST | 1 | P3 |
| `cma` | Cost & Mgmt. Accounting | 2 | P4 |
| `audit` | Auditing & Ethics | 2 | P5 |
| `fm` | Financial Management | 2 | P6 |
| `sm` | Strategic Management | 2 | P6 |

### 2.2 Chapter

Each subject has an ordered list of chapters (already populated in `SYL`). A chapter is identified by `subject + index`, e.g. `aa_3` = AS 1 chapter under Advanced Accounting.

### 2.3 Topic *(new)*

A chapter contains an ordered list of **topics** — finer than a chapter, used as the **bookmark** for "till what topic did I reach?" in partial sessions.

**How topics get populated:**
- We do **not** pre-populate topics for every chapter. Too much data, and ICAI structure shifts.
- Topics are added **on the fly** by the user. When ending a session, the user types a free-text "I reached till: ___". If that string is new for this chapter, it becomes a new Topic. If it matches an existing one, it's reused.
- A chapter's topic list is therefore the user's evolving table of contents.
- For convenience, the dropdown shows previously-used topics for that chapter, plus a "type new" option.

```
Topic = {
  id, subjectId, chapterIdx, name, order, createdAt
}
```

`order` = sequence number, so we know "topic 3 comes after topic 2." Auto-assigned when added; user can drag-reorder later.

---

## 3. Activity types

The seven activity types. Each one drives different fields in the session/quick-complete form.

| Activity | Code | What it is | Sibling of | Notes |
|---|---|---|---|---|
| Lecture | `lecture` | Watching/attending a lecture | — | Has total duration; can have homework attached |
| Homework | `homework` | Doing homework that came from a lecture | `lecture` | Linked to the lecture entry that spawned it |
| First Reading | `first-reading` | First read-through of chapter material | — | Generates a `FirstReadingChunk` (see §6) |
| Revision | `revision` | Re-reading already-first-read material | `first-reading` | Tied to a specific chunk + revision number (R1, R2, …) |
| Questions | `questions` | Practice questions from books / modules / RTPs | — | Scored by attempted/correct |
| Test | `test` | Formal mock / MTP / RTP / past-paper test | — | Has reading-time + test-time phases (see §5) |
| Flashpoint review | `flashpoint-review` | Going through saved flashpoints | — | Optional. Maybe not v1. ❓ |

**Phase-3 sessions are not a separate activity type.** A session done as part of the final-revision-plan is just a `revision` activity, tagged with the goal id. See §9.

---

## 4. The TimeEntry — the universal record

This is **the** record. Every action a student takes that gets logged becomes one of these. The Timeline tab is `SELECT * FROM TimeEntries WHERE date = X`.

```
TimeEntry = {
  id,                     // unique, used for sync merge
  createdAt, updatedAt,

  // ── WHEN ──
  date,                   // YYYY-MM-DD (calendar date the session occurred)
  startTime,              // HH:MM
  endTime,                // HH:MM (computed if duration mode)
  elapsedMin,             // actual clock time = endTime − startTime − pauseTime
  pauseMin,               // total pause time during the session (pomodoro breaks count here)

  // ── WHAT ──
  activityType,           // 'lecture' | 'homework' | 'first-reading' | 'revision' | ...
  subjectId,              // e.g. 'aa'
  chapterIdx,             // index into SYL[subject].chapters

  // ── PROGRESS (the time-≠-progress split) ──
  fromTopicId,            // bookmark at session start (null if first session of chapter)
  toTopicId,              // bookmark at session end
  contentMin,             // OPTIONAL — for lectures only. "Lecture content consumed."
                          //   If null, contentMin == elapsedMin.
  status,                 // 'complete' | 'partial' | 'in-progress'
                          //   'in-progress' means session was paused mid-flight, can be resumed
                          //   'partial' means user explicitly stopped before finishing
                          //   'complete' means user finished what they intended

  // ── HOW ──
  mode,                   // 'normal' | 'pomodoro' | 'quick-complete'
  pomodoro,               // { workMin, breakMin, longBreakEvery, longBreakMin } — null if mode != 'pomodoro'

  // ── OUTCOME (activity-specific payload) ──
  payload: { ... see §4.1 }

  // ── META ──
  notes,                  // free-text
  stuckNoteId,            // optional link to a stuck-point flashpoint
  flashpointIds,          // [array of flashpoints created during this session]
  goalId,                 // optional — links session to a goal (e.g. final-revision-plan pass)
}
```

### 4.1 Activity-specific payload

Each activityType reads/writes its own fields under `payload`:

| Activity | Payload fields |
|---|---|
| `lecture` | `lectureLabel` (free text, e.g. "Lec 7" or "Amalgamation Part 2"), `lectureNumber` (optional int), `plannedDurationMin` (the "this lecture is 2 hours"), `homeworkGiven` (bool), `homeworkText` (if homeworkGiven, free text describing what was given) |
| `homework` | `linkedLectureEntryId` (the TimeEntry id of the lecture this came from), `homeworkLabel` (free text) |
| `first-reading` | `chunkId` (which FirstReadingChunk this session contributes to — see §6) |
| `revision` | `chunkId` (which chunk is being revised), `revisionNumber` (1, 2, 3, …), `confidence` (1–5, optional) |
| `questions` | `source` (free text — book / module / RTP / etc.), `attempted`, `correct` |
| `test` | `source` (MTP / RTP / past paper / faculty / custom), `readingTimeMin` (actual time spent in reading phase), `testDurationMin` (actual time spent answering), `score`, `maxMarks`, `weakAreas[]` (array of free-text tags) |
| `flashpoint-review` | `flashpointIds[]` (which ones were reviewed) |

---

## 5. The three entry contexts

The single biggest mistake to avoid: asking for fields the user can't possibly know yet.

### 5.1 Schedule / Add (before doing it)

You're planning a future activity. You only know the activity type and what it's about — not how long it will take or how it will go.

| Activity | Fields asked when scheduling |
|---|---|
| Lecture | `lectureLabel`, `subjectId`, `chapterIdx`, `plannedDurationMin` (optional), planned date (optional) |
| Homework | `linkedLectureEntryId`, `homeworkLabel`, deadline (optional) |
| First Reading | `subjectId`, `chapterIdx`, planned date (optional) |
| Revision | *Not scheduled manually — auto-generated from chunks (see §6)* |
| Questions | `subjectId`, `chapterIdx`, `source` |
| Test | `source`, `subjectId` (or "all"), planned date |

**Notably absent at schedule-time:** start time, duration, status, score, score, homework time, homework text, weak areas, etc. None of those are knowable in advance.

### 5.2 Session (while doing it)

You're starting a live session right now. The app fills in everything it can automatically.

**Auto-filled by app:** `date` (today), `startTime` (now), `endTime` (when you stop), `elapsedMin` (live counter), `pauseMin` (sums pomodoro breaks).

**Asked at session start:** activity type, subject, chapter, mode (normal/pomodoro), pomodoro config (if pomodoro), `fromTopicId` (pre-filled to last bookmark for this chapter), and for revisions only — *which due/overdue chunk are you revising?*

**Asked at session end:** status (complete / partial), `toTopicId` (bookmark advanced to), the activity-specific outcome fields:
- Lecture → `homeworkGiven` + `homeworkText` (yes — these are asked **now**, not when scheduling), optionally `contentMin` ("how much of the lecture content did you actually cover?")
- Test → `score`, `maxMarks`, `weakAreas`, plus the test session has its own dual-phase flow (see §5.4)
- Questions → `attempted`, `correct`
- Revision → `confidence`

### 5.3 Quick Complete (after the fact)

You did something earlier and want to log it retroactively. You know everything that happened.

**All fields asked.** Same as session-start + session-end combined, plus you must enter `date`, `startTime`, and either `endTime` OR `durationMin`. Mode is forced to `quick-complete`.

### 5.4 Test session — special flow

Tests are the only activity with a structured multi-phase live timer:

```
[Reading Phase]   countdown from configured reading time (e.g. 15 min)
       ↓
[Test Phase]      countdown from configured test duration (e.g. 180 min)
       ↓
[Outcome Form]    score, max marks, weak areas, source
```

Both phases are optional-skippable. Both phases store their actual elapsed time independently in payload (`readingTimeMin`, `testDurationMin`). The total `elapsedMin` of the TimeEntry = readingTimeMin + testDurationMin.

---

## 6. First-reading chunks and revision schedule

### 6.1 What is a chunk?

A `FirstReadingChunk` is a contiguous range of chapter content that a student first read **on a given day**. Chunks are the unit that revisions are scheduled against.

```
FirstReadingChunk = {
  id,
  subjectId,
  chapterIdx,
  date,                    // the calendar day this chunk is anchored to
  fromTopicId, toTopicId,  // the range of content in this chunk
  status,                  // 'open' | 'closed'   — open means more sessions today might extend it
  sessionEntryIds[],       // TimeEntries that contributed to this chunk
  totalElapsedMin,         // sum of elapsedMin across contributing sessions
  totalContentMin,         // (lectures only — n/a here) — keep null
  revisions: [
    { number: 1, dueDate, status: 'upcoming'|'due'|'overdue'|'done'|'skipped', completedDate, entryId }
    { number: 2, ... }
    ...
  ]
}
```

### 6.2 Chunking rule ❓ (waiting for your A/B/C answer)

Provisional rule (Option B): **all first-reading sessions on the same `(subjectId, chapterIdx, date)` tuple form one chunk.** A new day = a new chunk.

When a first-reading session is logged:
1. Look for an existing chunk with same subject + chapter + date.
2. If found → append this session, extend `toTopicId`, recompute totals.
3. If not found → create a new chunk, schedule its revisions (R1, R2, …) using the user-configured offsets from setup.

A chunk closes automatically at end-of-day, or when the user marks first-reading complete for that chapter (no more chunks will be created for that chapter from then on, but existing chunks still get their revisions).

### 6.3 Revision schedule

Default offsets from setup screen: **+1, +7, +30, +60, +90 days** (already user-configurable in your existing setup UI). When a chunk is created on date D, each revision N gets `dueDate = D + offset[N]`.

State machine for each scheduled revision:
- `upcoming` — due date is in the future
- `due` — due today
- `overdue` — due date is past, not yet done
- `done` — completed
- `skipped` — user chose to skip (with reason note)

When the user starts a `revision` session, they pick from a list of revisions across all chunks where status ∈ {due, overdue, upcoming-soon}. Doing a revision marks that scheduled revision `done` and stamps `completedDate`. **Doing R2 does not require R1 to be done** — but the UI will warn if R1 is skipped.

### 6.4 Partial revision

If a revision session is `partial` (didn't reach the chunk's `toTopicId`):
- The session entry stores its own `fromTopicId → toTopicId`.
- The scheduled revision **does not advance to `done`** yet — it stays `due` / `overdue`.
- Next time the user picks this same revision, the form pre-fills `fromTopicId` to where they left off.
- Once the session reaches the chunk's full `toTopicId`, that revision is marked `done`.

---

## 7. Partial completion — full mechanics

### 7.1 The two numbers (recap)

| Number | What it is | Where it shows |
|---|---|---|
| `elapsedMin` | Clock time you spent. 1pm–2pm = 60 min, regardless of how much content. | **Timeline** (everywhere). Daily study totals. Streaks. |
| `contentMin` | Lecture content consumed. Optional, lectures only. 60 min elapsed but only 40 min of lecture played. | **Lectures tab** progress. "How much of this lecture is left?" |
| `toTopicId` | Where you got to in the chapter. | **Resume** button pre-fill. Lecture / chapter row "in progress: …last topic: X". |

### 7.2 The follow-up entry

When a session ends with `status = 'partial'`, the system **does not create a new TimeEntry** — instead, it sets a flag:

- For lectures: the parent **scheduled lecture** (or the lecture-tracker row) shows status `in-progress` with `contentMin / plannedDurationMin` and `toTopicId`.
- For first-reading: the chunk for that day stays `open`, and the chapter row in Syllabus tab shows `in-progress` with `toTopicId`.
- For revision: the scheduled revision stays `due` / `overdue`.

This avoids cluttering the timeline with placeholder entries. The "follow-up" is just the natural state of the parent thing being unfinished.

### 7.3 Where in-progress items surface

Every in-progress thing surfaces in three places:
1. **Today tab** — top "Resume" strip, in priority order (oldest in-progress first, or user-configurable).
2. **The owning tab** — Lectures tab shows in-progress lectures, Syllabus tab shows chapters with open chunks, Revisions tab shows partial revisions still due.
3. **Session tab** — when you pick activity + chapter, if there's an in-progress item for that combo, the form pre-fills `fromTopicId`.

---

## 8. Reference time (the no-stacked-countdowns rule)

When the user starts a session, show **one** "Reference: Xh Ym" line beneath the live timer. Picked smartly:

| Activity | Reference shown |
|---|---|
| First reading | None (it's the first time) |
| Revision (R1) | First reading total `elapsedMin` for this chunk |
| Revision (R2+) | Previous revision (R1 for R2, R2 for R3, …) `elapsedMin` for this chunk |
| Lecture | Planned duration (from schedule) |
| Test | Configured reading time + test time → these become **actual countdowns** because tests are by design timed |
| Questions | None (no obvious anchor) |
| Homework | None (or user-set target) |

Visual rule: live timer is white. Crosses 1.0× reference → amber. Crosses 1.5× → red. No second clock ever ticks down except for tests.

User can override and set a manual target before starting any session. Manual target replaces the smart reference.

---

## 9. Goals (the replacement for "phases")

A `Goal` is a user-defined target with a deadline. Goals run in **parallel**. The Dashboard shows every active goal as its own progress bar.

### 9.1 Goal types

```
Goal = {
  id, type, title, subjectIds[], targetDate, status, createdAt, completedAt,
  config: { ... type-specific }
}
```

| Type | What it tracks | Type-specific config |
|---|---|---|
| `lectures-complete` | All lectures of selected subjects | — |
| `first-reading-complete` | First reading of selected subjects (all chapters) | — |
| `question-bank` | N questions practiced | `targetCount` |
| `tests-attempted` | N tests done | `targetCount` |
| `final-revision-plan` | The final-month multi-pass planner | see §9.3 |
| `custom` | Free-form (count or duration based) | `targetMin` or `targetCount`, plus a tag |

Status: `not-started` | `in-progress` | `complete` | `overdue`.

### 9.2 Goal progress

Computed on the fly from TimeEntries — never stored. Examples:
- `lectures-complete` for AA → count of distinct `(chapterIdx, lectureLabel)` lecture entries with status=complete, divided by total lectures planned (❓ where do lecture totals come from? See open questions.)
- `first-reading-complete` for G1 → chapters in G1 subjects with at least one closed chunk reaching the chapter's last topic, divided by total chapters.
- `final-revision-plan` → see §9.3.

### 9.3 Final-revision-plan (the special one — replaces "Phase 3")

This goal type has its own planner UI because the math is unique. Anchored backwards from the **first exam date**, it generates a day-by-day schedule.

**Inputs the user provides:**
- First exam date.
- **Buffer days** — N days between the end of the last revision pass and the first exam (e.g. 2 days "I just want to chill / extra practice before exam day").
- **Number of passes** — e.g. 4.
- **For each pass:**
  - Days per subject (e.g. Pass 1 = 10 days/subject; Pass 2 = 7; Pass 3 = 3; Pass 4 = 1).
  - Subject order within the pass (drag-to-reorder).

**Output (auto-generated calendar):**
A list of `(date, subjectId, passNumber)` tuples covering the entire planner window. Working backwards from `firstExamDate − bufferDays`:
```
Pass 4: 1 day each × N subjects = N days
Pass 3: 3 days × N = 3N days
Pass 2: 7 days × N = 7N days
Pass 1: 10 days × N = 10N days
+ bufferDays
= total planner length
Planner start date = firstExamDate − totalPlannerLength
```

Each calendar day points to one subject. When the user does a `revision` session on that day, the subject is pre-selected and the session is auto-tagged with `goalId` of this plan + `passNumber`.

**Progress** = days-completed-on-schedule ÷ total planner days.

**Slippage handling:** if the user misses a day, the planner shows it as red and offers two re-plan options: (a) "compress remaining" — squeeze missed days into upcoming ones, or (b) "extend deadline" — push everything later (only if the buffer can absorb it).

---

## 10. Pomodoro

User-configurable, no fixed 25/5.

```
PomodoroConfig = {
  workMin,           // default 25
  breakMin,          // default 5
  longBreakEvery,    // default 4 (every Nth break is a long break)
  longBreakMin       // default 15
}
```

- A **default** is set in Settings.
- Each session can **override** the default at start time.
- During a pomodoro session, work intervals contribute to `elapsedMin` and breaks contribute to `pauseMin`. Both are recorded.

---

## 11. Flashpoints

Unchanged from your existing design, just formalized.

```
Flashpoint = {
  id, title, body, subjectId, chapterIdx, topicId (optional),
  createdAt, createdInEntryId (the session it was captured during, optional),
  reviewCount, lastReviewedAt
}
```

A flashpoint can be created from inside any session (a "capture flashpoint" button). Reviewing flashpoints is its own activity type (`flashpoint-review`).

---

## 12. Storage and sync (unchanged from v2, but formalized)

```
APP = {
  cfg,                    // user setup config (name, exam date, group, exemptions, revision offsets, pomodoro defaults)
  topics: [],             // all Topic records across all chapters
  timeEntries: [],        // all TimeEntry records — the spine
  chunks: [],             // all FirstReadingChunk records (derived but stored for revision-schedule integrity)
  lectures: [],           // scheduled lectures (the "add a lecture to track" entries)
  homework: [],           // scheduled homework backlog
  tests: [],              // scheduled tests backlog (separate from completed test TimeEntries)
  flashpoints: [],        // all Flashpoint records
  goals: [],              // all Goal records
  meta: { savedAt, version }
}
```

**Persistence layers** (already in v2 — keep):
1. `localStorage` — primary, every save.
2. File System Access API — optional, writes back to the HTML file itself.
3. GitHub Gist — optional, sync across devices.

**Sync fixes** (from earlier review — to be applied during build):
- Auto-pull from cloud on app startup before accepting any input.
- Dirty-flag re-queue if a change fires during an in-flight sync.
- 2 sec debounce on cloud writes.
- "Last synced X ago" tooltip in the topbar.

---

## ❓ Open questions (need your answers)

1. **Chunking rule** — Option A (per session), B (per day), or C (per chapter)? (I think you said B; please confirm with the letter.)
2. **Lecture totals** — for `lectures-complete` goal, how does the app know the total number of lectures for AA? Options:
   - (a) User pre-enters lecture list per chapter when adding a goal ("I have 47 lectures total")
   - (b) User adds lectures to a backlog as they go, and the goal denominator is "lectures added so far"
   - (c) User just sets a manual target count
3. **Flashpoint review** — is this a session activity in v1, or just a tab where you read them like notes (no time tracking)? My instinct is the latter for v1.
4. **Topic dropdown** — when ending a session, should the "till topic" dropdown also let you pick "I went past the end of this chapter" (to mark chapter complete)? Or is chapter-complete a separate explicit checkbox?
5. **Homework deadline** — homework backlog items: deadline optional or required?
6. **Goal — multiple subjects in one goal** — is that one goal with N subjects, or one goal per subject auto-grouped? My current model is one goal with `subjectIds[]`.

---

## What this spec does NOT cover (yet)

These get their own spec files once the spine is locked:

- `02-today.md` — Today tab spec
- `03-session.md` — Session tab spec (live + pomodoro + test flow)
- `04-quick-complete.md` — Quick Complete forms per activity
- `05-syllabus.md` — Syllabus / first-reading tab
- `06-lectures.md` — Lectures tab + lecture backlog
- `07-revisions.md` — Revisions tab (due / overdue / upcoming)
- `08-tests.md` — Tests tab
- `09-flashpoints.md` — Flashpoints tab
- `10-dashboard.md` — Dashboard tab + all stats it shows
- `11-timeline.md` — Timeline tab
- `12-goals.md` — Goals tab + final-revision-plan UI
- `13-settings.md` — Settings tab + sync UI
- `14-help.md` — Help / onboarding
