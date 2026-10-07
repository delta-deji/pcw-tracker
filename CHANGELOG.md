# Changelog

## 0.1.0 - 2026-10-07

Phase 1 initial build.

- Skill skeleton, trigger phrases and slash command (`build|refresh|update|scope`).
- Compute spec covering counting, buckets, scope/drift, dates, pace, signals, forecast, Needs
  attention, dependencies, stale work, reconciliation, data confidence, commitments, snapshots.
- Config schema.
- First-time walkthrough script.
- Page template and capability wiring (`db` + `user`, owner-only writes).

No defaults changed yet via the end-of-session question - none asked yet, first real walkthrough
pending (PCW-1311).

## 0.1.1 - 2026-10-07

First real build (PCW-1311) and page redesign, same day.

- Ran the full walkthrough live against PCW-1311: scope (4 of 9 epics), status mapping, reporting
  start, commitment (30 Oct), share/schedule, defaults.
- Caught and fixed a scope-transparency bug: an untracked epic whose Jira status was Blocked
  (not Won't do) was wrongly excluded from the "still open" count.
- Discovered a second Jira connector gap beyond the `parent`-field one from Phase 0: the standard
  `duedate` field is dropped by `searchJiraIssuesUsingJql` entirely, in every view and field
  combination tried. Only `getJiraIssue(view:"full")` returns it. Logged in `SKILL.md` site facts.
- Rebuilt the published page from scratch after reviewing the brief's actual visual reference
  (previously skipped - see `templates/page.html` for the approved base template): blue-accent
  headline card, segmented progress bar, collapsed/expandable epic rows, a real Gantt with a date
  axis and today/commitment lines, grouped task-activity sections, compact dependency rows.
- Added an early-pace caveat (`compute-spec.md` §6 addendum): when there's under 7 working days of
  real activity history, flag run rate and forecast as provisional rather than let a thin-data
  number read as a flat alarm.
- Moved the "untracked epics still open" notice from the page header down to sit with the
  collapsed Not Tracked section at the bottom - the header should read as focus on what's tracked,
  not open with a disclaimer about what isn't.
- Dropped "Updated last working day" from Task Activity (not useful; the focus is Stale, which
  forces a conversation about what's stalling).
- Fixed ~140 em dashes across every skill file and the page itself - house style is hyphens only.

## 0.1.2 - 2026-10-07

Second round of page feedback, same day.

- Header restructured: `<h1>` is now the PCW's own title + " Delivery Tracker" (e.g. "Extras -
  Bonus Content Delivery Tracker"), with the PCW key and scope line underneath. Previously the key
  was the heading and the title was the subtitle - the wrong way round.
- Dropped "Time elapsed vs work done" entirely (`compute-spec.md` §6) - it duplicated the headline
  % and added nothing next to it.
- Forecast finish is now suppressed while pace confidence is low, not just caveated - shows "available
  once there's a full week of run-rate history" instead of a date. A caveated-but-still-shown number
  still read as a flat alarm with only 3 days of history behind it.
- Section titles (Needs attention, Gantt, etc.) restyled with full-contrast text and an accent
  underline so they read as headers, not body copy.
- Gantt reworked for date clarity: a shared weekly gridline overlay so every epic's bar lines up
  with the axis, explicit "start to due" text next to each epic (not just a hover tooltip), and the
  Gantt formally pulled forward into v1 at Deji's request (was slated Phase 2 in the original
  brief).
- Self-review caught two real spec gaps, not just cosmetic ones: the headline pill row was missing
  Blocked entirely (checklist had wrongly marked this item as passed), and Excluded/Unmapped counts
  weren't shown beside the pills at all, only buried in the footer. Also fixed every epic-key link,
  which pointed at a dead `#` anchor - they now open the real Jira issue.
- The Excluded/Unmapped fix above over-corrected: giving them their own pill-styled span next to
  the colourful bucket pills "creates noise" (Deji's words). Moved to a quiet trailing clause on
  the already-muted label line above the pills, shown only when non-zero.

## 0.1.3 - 2026-10-07

Got live visual access to the rendered page for the first time this build (Deji signed in to the
browser pane) - found a real bug no amount of source-reading would have caught.

- **In PR pill and progress-bar segment were rendering completely unstyled** - no dot, no colour,
  plain text. Root cause: `inPR`/`inQA` buckets were wired to colour names ("blue"/"purple") whose
  CSS (`.pill.blue`, `--blue-bar`, `--purple-bar`) was never actually defined. Added both, fixed in
  light and dark mode, confirmed visually.
- **"Signal" relabelled to "Status"** - the value (Critical/On track/Under pressure) was already
  plain English, but the bare label above it meant nothing on its own ("Signal means nothing!").
  Footer explainer rewritten as full sentences instead of a formula.
- **Epic run rates row alignment fixed**: numbers now align with the top line of a wrapped epic
  title instead of floating at the vertical centre of 2-3 lines of text, which read as arbitrary.
  Added a proper "Left / Progress / % / Time left / Status" header row, on a shared CSS grid with
  the data rows so the columns actually line up (a flex layout with `margin-left:auto` can't
  guarantee that).

## 0.1.4 - 2026-10-07

First real `refresh` (not `update`) against live Jira, same day. Config untouched, as designed.

- Detected real drift: VC-4038 moved from In PR to Ready for QA since the last compute, about 90
  minutes earlier. Correctly bucketed into In QA (the status mapping already covered "Ready for
  QA" from setup, so this needed no new mapping decision).
- Added a page section for this: Task Activity previously only ever showed an "In PR" group
  because In QA had been empty since setup - now shows "In QA" too, conditionally, the first time
  there's actually a task in it.
- Re-checked all four epics' due dates via the per-epic full-view fetch (the walkthrough now
  requires this explicitly, see §1) - none had moved. Re-checked for new child epics - none.
- Headline numbers otherwise unchanged (same % complete, same run rate - no new Done transitions
  in the 90 minutes since the last compute). Today's snapshot document updated in place (version 1
  to 2), not duplicated, confirming compute-spec §7's same-day-replace rule works as designed.

## 0.1.5 - 2026-10-07

First real `/pcw-tracker scope` run - deliberately exercised to clear this skill's own Phase 3
promotion gate ("run cleanly on a real PCW and survived a config change").

- Discovery found nothing drifted (same 9 child epics, no unmapped statuses) - reported that
  plainly rather than treating "nothing changed" as nothing to say.
- Proactively re-asked about ROKU-1967's exclusion, since its original Phase 0 reason
  ("placeholder") predates the other epics gaining real dates and activity. Deji's call: keep it
  excluded, same reason.
- That's still a real config write, not a no-op: added a `reconfirmations` entry recording when
  and how the exclusion was revisited, per a new field documented in `config-schema.md`. Config
  version bumped 1 to 2 in the artifact's database.
- `/pcw-tracker scope`'s own instructions (`walkthrough.md`) updated to make this the expected
  pattern going forward: a clean "nothing new" result is still reported, and revisiting one
  existing decision whose premise may have changed is fair game, not scope creep.
- This clears the "survived a config change" half of the Phase 3 promotion gate. The "checked
  `DiceTechnology/deltatre-ai-tools`'s contribution conventions" half is still outstanding.
