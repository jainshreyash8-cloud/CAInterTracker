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
First Reading + day-grouped chunks + Syllabus tab (per the data-model rules above).
