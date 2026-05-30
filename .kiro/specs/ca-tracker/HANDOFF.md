# CA Inter Tracker — Handoff / Locked Decisions

> Read this first. It captures every decision so a new session has full context without replaying chat history.
> Owner: Shreyash Jain. Repo: `jainshreyash8-cloud/CAInterTracker`.

## What we're building
A **single offline HTML file** study tracker for CA Inter students, with optional **GitHub Gist** cloud sync (bring-your-own-token, zero infra on our side). Works fully offline via localStorage; syncs across devices when online.

## Working method (IMPORTANT — avoids past loops)
- Build **incrementally, one working flow at a time**. Never big-bang the whole app.
- Separate **look** (cheap, static) from **logic** (wire one flow, user tests, lock, next).
- Each slice = one branch + one preview URL via `https://raw.githack.com/jainshreyash8-cloud/CAInterTracker/<branch>/<file>`.
- Do NOT re-litigate locked decisions below.

## LOCKED design decisions
- **Bento-card design language everywhere.** No colored-left-outline cards (user hates them).
- **Brand:** "Tracker" in **Sora, weight 800, letter-spacing -.03em**, with "by Shreyash Jain" subtle below in JetBrains Mono. NOT italic.
- **Theme:** DARK confirmed working; LIGHT version also built. (User to give final DARK-vs-LIGHT call — check latest message.)
- **Width:** "padded" (centered, max 1280px) is current default. User may switch to full-width edge-to-edge — confirm.
- **Icons:** clean inline line-icons (stroke, currentColor). NO emojis.
- **Numbers:** one baseline per stat row; big value + small unit (e.g. `3` + `h` + `20` + `m`), no mid-number size jumps.
- Fonts: Sora (brand/headings/numbers), Instrument Sans (body), JetBrains Mono (labels/meta).
- Prototype (look-only, 3 tabs) lives at `prototype.html` on branch `build/prototype-look`.

## LOCKED data-model / behavior decisions
- **Time is the schema.** Every activity = one TimeEntry (date, start, end, elapsed). Timeline = view of these. Dashboard = aggregates.
- **Status = only two:** `done` or `in-progress`. NO "stuck", NO separate "partial". In-progress remembers where you stopped + minutes done + a free-text "what I covered" note.
- **Time ≠ progress:** store `elapsedMin` (clock time, drives Timeline) AND for lectures `contentMin` (actual content consumed). Plus a free-text "stopped at ___" bookmark. NO structured topic lists (too many topics; user rejected dropdowns).
- **No overlap:** two time entries can NEVER share the same time window. Hard block on save (live + quick-complete both validate).
- **Lectures live INSIDE chapters** (expandable chapter sections, each with main + revision-lecture sub-rows). Add lectures per chapter as received. NO recorded/live mode toggle. NO manual lecture counts — totals are computed.
- **Revision lectures** = separate activity from main lectures (ad-hoc YouTube/refreshers); can be added inside chapters too.
- **Misc/Bonus pseudo-chapter** per subject for orphan lectures (still counts toward hours).
- **Lecture-completion due date** for a whole subject lives in the Lectures tab (this is the only "phase 1 end" — NOT a setup-screen field).
- **First reading chunks grouped by DAY** (option B): all first-reading sessions on same chapter same calendar date = one chunk; new day = new chunk. Each chunk schedules its own revisions.
- **Revision schedule offsets** default `1,7,30,60,90` days (editable in setup AND settings; add/remove; lay out evenly so it looks good).
- **NO "phases" feature.** Only a **consolidated final-revision planner**: default **3 passes of 7/3/1 days per PAPER**; combined-subject papers (P3=IT+GST, P6=FM+SM) split each pass half-half (3.5/1.5/0.5); user sets pass count, buffer days (default 2), and **per-paper order with drag-to-reorder**. Anchored backwards from first exam date. Reorder is per-PAPER not per-subject. Planner must let you START sessions from it.
- **Exam-countdown card on Today expands** to show current phase + the planner inline (no separate phase card).
- **Goals feature: dropped.** Dashboard auto-progress only.
- **Pomodoro:** fully configurable (work/break/long-break-every/long-break-min); saved default in Settings + per-session override. Breaks do NOT count as study time (`elapsedMin` excludes pauses + pomodoro breaks).
- **Exemptions:** two types — Standard (3-attempt validity; need 40% each remaining paper + 50% aggregate) and Permanent (need 50% each remaining paper). Per-exempted-paper picker in setup + settings. Tests tab uses this to show marks-needed-to-pass vs averages. Active-subjects logic must respect exemptions everywhere (aggregations, planner paper count).
- **Chapters editable** (add/edit/reorder/delete; reset to ICAI default) per subject in Settings. `chapList()` must return a COPY when falling back to the SYL constant (never mutate the constant).
- **Quick Complete** available in many places (Session tab top, each lecture row, each chapter row in Syllabus, each revision row). Always asks date + start time + end time (or duration) + per-activity fields. Must respect no-overlap rule.

## Tests tab specifics (for when we build it)
- 3 types kept separate in stats: **Full-syllabus** (MTP/Model/PYP — default 15 min reading + 180 min test, both editable), **Chapterwise/sectional** (all fields configurable), and **RTP is NOT a test** — it's a question bank → goes to a Questions session.
- Test session = **real countdown** (reading phase then writing phase), NOT count-up. Only **Complete** or **Abort** (no partial/stuck). Abort → time still logged to timeline, test reappears as un-attempted. Complete → goes to review.
- Guard against divide-by-zero when maxMarks=0; use `?? default` not `|| default` so 0 is accepted.

## Questions activity (simple model)
- Fields: source (textbook/RTP/MTP-old/faculty/AI-generated/mixed), chapter(s) or "mixed", optional rough count. Capture flashpoints during. NO attempted/correct scoring.

## Flashpoints (for when we build it)
- Per-subject sub-tabs. Add via a title+point row format (two oval boxes: title, then the point), with "add row" for multiple. When added from a session (lecture/reading/revision/questions) subject+chapter auto-filled. NO source field. Read like notes (not a timed activity). Nothing is "optional".

## Robustness must-haves (from user's bug review)
- localStorage quota: try/catch + warn near limit + offer export.
- Debounce/queue file + cloud writes; cloud sync = session-level merge by id (not overwrite); auto-pull on startup; dirty-flag re-queue; "last synced" shown.
- Streak ignores sessions < 5 min.
- Heatmap/date ranges clamped (max ~18 months) + validate dates on entry.
- Long-running session: at 8h, PROMPT "what time did you actually finish?" — never silent-delete.
- Cap pace-ratio projections (one mis-logged 10h lecture must not skew estimates).
- Escape ALL user input via esc() before innerHTML (prevent layout-break).
- Mobile: save timer state every tick + on `pagehide`/`visibilitychange` (not just beforeunload).

## Repo state  (IMPORTANT — read carefully)
- **Everything important now lives on `main`.** Files on `main`: `README.md`, `ca-tracker-v2.html` (original, reference), `prototype.html` (look-only v3 design — open via raw to preview), and this `HANDOFF.md`.
- **ENVIRONMENT QUIRK:** in the sandbox, pushes to non-default branches report "success" but do NOT reach real GitHub — only pushes to `main` actually land. So: **commit build work to `main` and push `main`** (or verify any branch actually appears on GitHub before relying on it). The old `build/prototype-look` and `build/slice-0.1-lectures` branches and "PR #1" never reached GitHub — ignore them; their content is preserved in `main`'s history.
- The repo is **private**, so raw.githubusercontent / raw.githack URLs return 403 unless the repo is made public or accessed with auth.

## NEXT STEP for new session
Start building the real working app as `index.html` on a fresh branch (e.g. `build/v1`), incrementally:
1. Foundation: tokens/CSS from prototype v3, app shell, setup screen (name, exam date, group, exemptions w/ types, revision offsets), storage (localStorage), Gist sync skeleton with the fixes above.
2. Lectures (inside-chapters) + Session (lecture + revision-lecture) end-to-end with timer/pomodoro/pause/resume, no-overlap guard, in-progress resume.
3. First reading + chunks + Syllabus tab.
4. Revisions queue + revision sessions.
5. Tests (dual-phase countdown).
6. Questions + Homework + Flashpoints.
7. Dashboard + final-revision planner.
8. Settings polish + Help + sync hardening.
Test each step via raw.githack preview before moving on.



## Build progress (updated 2026-05-29)

`index.html` on `main` is the REAL working app. Done so far:
- Foundation: setup (name/exam/group/exemptions Standard+Permanent/revision offsets), dark+light themes, padded/full-width toggle, bento design, Sora brand, line-icons.
- Lectures tab: lectures live inside chapters (expandable), main + revision lectures, add/edit/delete, Misc/Bonus chapter, computed totals, per-subject completion due date.
- Session: lecture + revision-lecture via unified subject -> chapter -> lecture DROPDOWN + mode (live/pomodoro) + Start; drift-free timer; pause/resume; fullscreen; end as done/in-progress with "minutes of lecture actually completed" + "where I stopped" note; links back to the lecture row (shows partial % + note + Resume).
- Quick-complete with hard no-overlap guard. Today (bento stats, expandable exam plan, readiness, week chart, resume strip). Settings, Help.
- Storage + Gist sync (auto-pull on startup, debounce, dirty-flag requeue, merge-by-id, last-synced).
- Stubs: syllabus, revisions, tests, flashpoints, dashboard.

## PENDING FIXES (do these FIRST in next session)
1. ~~Remove the "Source" field from the Revision Lecture session config.~~ **DONE 2026-05-29** (commit `fix(session): remove useless Source field…`). Removed the `cf-src` input + its wiring, `_sPick.source`, `source` on `APP.active`, the `payload.source` write, and the source display on the running-session screen. Hint reworded to "leave it as ad-hoc".
2. ~~Timeline as a visual day × hour grid.~~ **DONE 2026-05-29** (commit `feat(timeline): rebuild as visual day x hour grid`). Rebuilt `renderTimeline` as a day-by-hour grid using new `.tl-*` classes (themed to CSS vars for dark+light). Confirmed look with user before building:
   - Rows = **every calendar day in range**, newest first, empty days shown as blank rows (capped 90).
   - **Adaptive** hour window — widens to cover the earliest/latest session so nothing is clipped.
   - Bars placed by start time, width = real study time (`elapsedMin`), colored by subject. **Complete = solid fill; in-progress = faded + dashed border** (user picked the faded/dashed option).
   - **Click a bar → details popover** (activity, chapter, time range, duration, status, note) with a **Delete** action.
   - Right-hand **day total** colored by a **user-adjustable** target (inputs on the Timeline page, persisted in `cfg.dayGoalH` / `cfg.dayOkH`; defaults green ≥8h, amber ≥6h, dim below).

## NEXT SLICE (in progress)
First Reading + Syllabus — **DONE 2026-05-29**, then **redesigned** after user feedback (commits `feat(first-reading + syllabus)…` then `refactor(syllabus): rebuild as per-chapter mastery table`).

**First Reading activity:** real session activity. Config = subject → chapter → timer → start (no lecture step; tracked at chapter level). End form = note only (no content-min, no homework). Added to Quick-complete.

**Syllabus tab = per-chapter MASTERY TABLE** (this corrects the first attempt, which wrongly made day-chunks the centerpiece). Per subject: tabs + summary bento (first reads done %, reading time, avg confidence, not started) + a legend, then one row per chapter showing: status badge (done/in-progress/not-started) · first-read/started date · reading time · **confidence stars (user-set)** · **revision dots** (1/7/30/60/90, overdue=red / due=amber / upcoming=hollow). Expand a row → revision schedule with dates, day-by-day reading log, saved notes. Per-chapter actions: Start/Continue/Read-again reading, Mark-first-read-done / Reopen, **Notes** (summary-notes modal), Quick-log. Misc/Bonus excluded.

**Data model:** new `cfg.chData` keyed `sid_idx` storing only what time can't derive — `{conf, notes, firstReadDone}`. Status, first-read date, and reading time stay derived from `first-reading` time entries (time-is-schema preserved). `getCh()` is read-only (no empty-record bloat); `getChW()` writes.

### LOCKED revision model (decided with user — overrides the older "day-chunks schedule their own revisions" note)
Two layers:
1. **"Review yesterday" (daily consolidation, for the 24h forgetting curve):** any day you do **lectures, first-reading, or question practice**, the next day shows a one-off "review what you did yesterday" nudge (shows ~2 days then fades). **NOT** for revisions (revision already *is* review).
2. **Chapter spaced revision (long-term):** one R1–R5 series **per chapter** (NOT per chunk), counted from the day you **finish first reading** that chapter (`firstReadDone`), using `cfg.revOffsets` (default 1/7/30/60/90). **Lectures do NOT start this clock** — only first reading does (lectures are a firehose you only consolidate next-day; first reading is where you digest + make summary notes). Syllabus and Lectures tabs stay deliberately disconnected.

In this slice the revision dots are a **read-only preview**. The actionable due-revisions queue **and** the daily "review yesterday" nudge are the NEXT slice.

## NEXT SLICE (after this)
Revisions queue + revision sessions + the daily "review yesterday" nudge — **DONE 2026-05-29** (commit `feat(revisions): two-layer revision queue + revision sessions`).

- **Revision is a session activity** (subject → chapter, no lecture step) — but it is **NOT shown in the Session activity picker** (`ACT.revision.pick=false`). Revisions are scheduled by the app, so you launch them **queue-first** from the Revisions tab / Today nudge, never by blindly picking "Revision" then hunting for a chapter (user feedback: that was counterproductive).
  - Each queue item (due + review-yesterday) → tap **Start** → small **Live / Pomodoro** chooser → drops straight into the normal running-timer screen (same place all sessions run; toolbar chip + fullscreen unchanged). Functions: `launchRevision(sid,ci)` → `beginRev(sid,ci,mode)`.
  - **"Revise off-schedule"** button (`offScheduleRev()`) on the Revisions tab covers the rare ad-hoc case (pick subject+chapter+timer).
  - Guarded so you can't start a revision while another session is active.
  - Finishing it as Complete ticks off the chapter's next due revision; a quick **✓ Done** button on each queue item does the same without a timer. Completion stored in `chData.revs` (keyed by R number → date) and shown as green dots in the Syllabus table. Quick-complete also supports Revision (for logging past ones).
- **Revisions tab** has the two locked layers:
  1. **Review yesterday** — chapters with a lecture / first-reading / questions session in the last ~2 days that haven't been studied again today. Fully derived from entries (no extra state); fades after 2 days; clears once you study the chapter again. "Review now" starts a revision session.
  2. **Revisions due** — chapter spaced revisions from `firstReadDone` + `revOffsets`, **sequential + fixed dates** (only the next undone R is active; due/overdue first), plus a "coming up this week" peek. Decision locked with user: fixed schedule, done in order.
- **Today page** shows a "X due / Y to review" nudge linking to the Revisions tab.

## NEXT SLICE (after this)
Tests (dual-phase countdown) — per the "Tests tab specifics" section above. OR Questions + Flashpoints, then Dashboard + final-revision planner. Confirm order with user.


## Tests slice — DONE 2026-05-29 (commit `feat(tests): dual-phase countdown tests slice`, on `main`)

Rebuilt cleanly after the previous session's uncommitted Tests code was lost. `index.html` now has a fully working Tests tab.

- **Data:** `APP.tests=[]` (in freshState), each `{id,type:'full'|'chapter',source,name,subjectId,chapterIdx,readingMin,testMin,maxMarks,attempts:[],createdAt,updatedAt}`. Attempt `{id,date,startTime,endTime,durationMin,readingMin,marks,maxMarks,note,status:'completed'}`. `APP.activeTest` (separate from `APP.active`) holds the running test. `tests` added to the cloud-merge replace list. `ACT.test` = `on:true, pick:false` (never in the session picker — launched from the Tests tab, like Revisions).
- **Tab:** bento (tests count / attempts / avg % / not-attempted) + Full-syllabus and Chapterwise sections. Each test = expandable card → chips (source / reading+test mins / max marks), attempt history (date · marks · % colour-coded · duration · note · delete), and actions Start / Log past attempt / Edit / Delete. New/Edit via `testForm`→`saveTest` (defaults: full 15+180, chapter 0+90, 100 marks; `tf-seg` type toggle swaps source/chapter fields + prefills).
- **Dual-phase countdown engine** (`startTest`→`startTestLoop`/`testTick`/`paintTest`/`testPhaseLeft`/`clearActiveTest`, timestamp-based so it's drift-free + survives reload): reading phase then writing phase, big `.t-display` countdown coloured by phase (reading=accent, writing=ink→amber≤5m→rose at 0). `testSkipReading()` = "Start writing now". Only **Complete** or **Abort** (no pause — real-exam conditions).
  - `completeTest`→`finalizeTest`: marks (optional, clamped 0..maxMarks) + note → pushes a `completed` attempt AND logs a `test` time entry (status done, payload{testId,testName,type,source,marks,maxMarks,readingMin}). Divide-by-zero guarded via `!=null` checks + `maxMarks>0`.
  - `abortTest`: logs the time entry only (payload.aborted=true, no attempt) so the test stays un-attempted; confirms first.
  - `quickTest`/`submitQuickTest`: log a past attempt (pick test + date/start/end + marks + note) with the no-overlap guard.
- **Wiring:** `showPage` tests→`renderTests` (renders the running screen if `activeTest`). `wireShell` tb-timer click → `showPage('tests')` when a test is active, else `openFS()`. `init()` resumes `startTestLoop` if `activeTest`. `startSession` guarded against an active test (and `startTest` guarded against an active session). `entryTitle(e)` helper shows test name + marks in Today's list, the Timeline tooltip and the timeline block popover.
- Verified: `node --check` on all 18 script blocks + a logic smoke-test (scoring guard + phase math) passed.

### STILL TODO (unchanged, confirm look with user first)
Questions (session activity, no tab) · Flashpoints tab · Dashboard tab + final-revision planner. Per user's standing request, CONFIRM the exact Flashpoints + Dashboard look before building (adapt from v2 `renderDash`/`renderFlashpoints` but drop "phases", no emojis, use our data model).


## Tracker COMPLETE 2026-05-29 — Tests refinements + Questions + Flashpoints + Dashboard (all on `main`)

All tabs now functional. Commits: `fix(timeline)` (edit/delete popover double-close), `feat(tests+questions)`, `feat(flashpoints+dashboard)`.

### Bug fixed
- Timeline popover (`tlBlock`) Edit/Delete were broken: handlers called `close()` then returned undefined, so the modal framework auto-closed AGAIN and wiped the freshly-opened dialog. Fix = return `true` from those onClicks (the new modal replaces modal-root). Pattern note for future: any modal action that opens another modal must `return true`.

### Tests (final shape)
- `APP.tests[]`: `{id,type:'full'|'multi'|'chapter',source,name,subjectId,chapterIdx,chapterIdxs,dueDate,readingMin,testMin,maxMarks,attempts:[]}`.
- Attempt: `{id,entryId,date,startTime,endTime,durationMin,readingMin,marks,maxMarks,note,status:'awaiting'|'scored'}`. (Aborts create NO attempt — time-only entry.)
- 3 types: full (whole subject), multi (multi-select 2+ chapters via `chapterIdxs`), single chapter. Type LOCKED when editing.
- Optional `dueDate`. `testsToSit()` = no attempts + due≤today; `testsAwaiting()` = has an `awaiting` attempt. Both shown as Today nudges (`.today-due` strips) and in the Tests bento.
- Review-later flow: live **Complete** logs time + creates an `awaiting` attempt + entry (NO marks prompt), then auto-opens `enterResult`. `enterResult`/`saveResult` set marks+takeaways, flip to `scored`, and patch the linked timeline entry's `payload.marks`/`coveredNote`. `attPct` returns null when marks==null (awaiting excluded from best/avg; real 0 still counts).
- Helpers: `testChapIdxs`, `testLoc`, `pendingReview`, `testHasAttempt`, `testGroup`.

### Questions (session activity, no tab)
- `ACT.questions.on=true`. Config (`renderConfig` isQ branch) adds a Source select (`QSOURCES`=Textbook/RTP/MTP/PYP/Faculty/AI/Mixed) and a chapter dropdown with a "— Mixed / multiple —" option (default). `canStart` only needs a subject when mixed. End form adds an optional question-count field. Payload: `{source,mixed,count}`. Display via `qLoc(e)` in `entryTitle`, entryRow meta, timeline tooltip + popover. No scoring.

### Flashpoints (`renderFlashpoints`)
- `APP.flashpoints[]`: `{id,subjectId,chapterIdx,title,point,date,source}`. Per-subject filter chips (`.stab`) + live search (`fpSearchInput` re-renders + refocuses). Masonry `.fp-grid` cards with edit/delete. `addFlashpoint(pre)` = multi-row modal (`fpAddRow`/`fpRowHTML`, `.fp-rowx`), `saveFlashpoints` bulk-inserts. Edit modal ids are `fpe-*` (avoid colliding with end-form `ef-*`). `+ Flashpoint` button in the Timeline popover passes `{subjectId,chapterIdx}` to auto-fill.

### Dashboard (`renderDash`, adapted from v2 — NO phases, NO emojis, our data model)
- Helpers: `dashEntries` (active-subject entries w/ time), `studyDays`, `dataStart`, `totalStudyMin`, `dayMin`, `subProg` (first-read done/total), `overallPct`, `weakChapters` (read + conf 1–3, asc), `dbar` (CSS bar), `buildHeatmap` (CSS calendar grid, green intensity by daily hours, today ringed).
- Layout: b5 hero stats (overall % / days studied+missed / streak / total+avg hrs / exam countdown w/ avg test score) → heatmap card → 2-col `.dash-grid`: [activity breakdown bars, hours-by-subject bars] | [subject progress by group, recent sessions, weak chapters]. No canvas line chart (heatmap covers daily-hours-over-time; matches the agreed spec).

### Verify
`node --check` per `<script>` block (wrap each in IIFE — strict mode is per-script in the browser, so don't concat into one program). 20 script + 9 style tags, balanced.

### Possible follow-ups (not requested yet)
Final-revision planner (the old v2 "phase 3" replacement) is still not built. Flashpoint auto-capture currently only from the Timeline popover (could also add from session end / syllabus rows).


## Feedback round 2026-05-30 (commit `fix+feat: dashboard overflow, single-attempt tests, ...`)

- **Bug fixes:** dashboard left-cut (`.dash-grid>div{min-width:0}`); revisions "due today" tag stretched (`.rev-acts{align-items:center}`); removed "phase 1" wording in `editDue`.
- **Tests are now SINGLE-ATTEMPT** (user: once you've seen a test, re-sitting is pointless). `startTest` blocks if `testHasAttempt`; `testCard` hides Start + "Log result" once an attempt exists; `quickTest` only lists un-attempted tests. Delete the result (trash on the attempt row) to re-sit.
- **No instant marks prompt** on Complete (user: it's a 3h written test, marks come after checking). `completeTest` just logs time + creates an `awaiting` attempt; user hits **Enter marks** later. (Removed the auto-open setTimeout.)
- **Lecture pace & projection** (`_curPace` in Lectures): pace = (sum elapsedMin of `lecture` entries for subject) / (content min done: done→plannedMin, else contentDoneMin). Card shows `X× content time`, projected hours to finish remaining content, and /day to hit the due date. Per-lecture "~Xm at your pace" in `lecRow`.
- **Dashboard line chart** (`buildLineChart`): SVG, daily hours, total line (ink) + per-subject lines (subject color), viewBox 760×240 responsive. Added after heatmap.
- **Flashpoints importance**: `importance` 1/2/3 (normal/important/critical) → colored left border + tag, "★ key only" filter, importance-sorted. Add-modal rows redesigned to stacked title+importance+textarea (fixed the mismatched-box look). Edit modal has importance too.

### Phase clarification (for the record)
v2 had 3 "phases"; this build has NO phases at all. The only leftover wording ("phase 1" in the lecture due dialog) is now removed. A dedicated "final-revision planner" is still NOT built (optional future).

### Suggested-but-not-built (awaiting user pick)
Data insights: projected exam-readiness %, per-subject pace, best study time-of-day, consistency/streak analytics, test score trend, revision-overdue load. Flashpoints: tags/types (formula/trap/concept), pin-to-top, "quiz me" shuffle, export.



## Final Revision (consolidated) tab — BUILT 2026-05-30 (commit `feat(final-revision): consolidated revision planner tab`, on `main`)

New nav tab **"Final Rev"** (`pg-finalrev`, route `finalrev:renderFinalRev`). Replaces the old "phase 3" idea.

- **Config** `APP.finalRev = {rounds:[{id,len}], buffer, order}` (default 3 rounds 7/3/1 days-per-subject, buffer 2). User edits rounds (add/remove + days each) + buffer via `frEdit`. Exam date = `cfg.examDate` (set via `frSetExam`/`frEdit`).
- **Schedule = computed, never stored** (`frPlan`): total = Σ(round.len × #subjects); lastStudyDay = exam − (buffer+1); start = lastStudyDay − (total−1); blocks laid forward per round→subject. Combined papers appear as separate subjects (it/gst, fm/sm) since faculties differ — so no fractional-day split. `startsInPast`/`overDays` → amber "doesn't fit" warning.
- **Tracking** `APP.frTrack[roundId_sid_idx] = {done:date|null, min, skip, note}`. A chapter row has Revise (▶), done toggle (○/✓), note, skip (–/+). **Revise launches a real revision session** via `frRevise`→`launchRevision`; `beginRev` tags `APP.active._fr={roundId,sid,idx}`; `finalizeSession` then adds `elapsedMin` to `frTrack.min` and sets `done` on status==='done' (time-is-schema; shows on Timeline as normal `revision`).
- **Carry-forward**: each chapter shows previous round's time + note (`frPrev`) so later rounds go faster.
- **Today guidance**: in-tab `today-due` strip "Today → Round X · Subject" + Revise (first pending chapter). (Not yet added to the Today tab itself — possible follow-up.)
- UI reuses `.tst-card` collapsible blocks (one per subject) + `.stabs` round tabs + new `.fr-*` chapter rows.
- Added `finalRev`,`frTrack` to freshState + cloud-merge list.

### Still open / not done from the broader discussion
- **CSV import** (lectures/flashpoints/tests/chapters) — agreed, NOT built yet.
- **Syllabus ↔ flashpoint link** (chapter shows flashpoint count + ＋) — agreed, NOT built yet.
- **Flashpoints redesign to chapter-grouped collapsible list** (+ ★ binary flag, type tag, pin, export) — agreed earlier, NOT built yet (current flashpoints = card grid + post-hoc 2-dot importance).
- **Lectures card 4 → "Daily target"** (watch X/day to hit due date; tap still sets date) — agreed, NOT built yet.
- Dashboard insights (readiness/trend/debt/best-time/pace-on-dashboard) — **scrapped by user.**
