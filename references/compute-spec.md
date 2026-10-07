# Compute spec

The single source of truth for every number the tracker shows. If the page or the refresh logic
disagrees with this file, this file is wrong and needs fixing - don't let behaviour and spec
drift apart. Cross-referenced by `references/test-checklist.md`, which maps each rule below to
where it's implemented.

Dates throughout are calendar dates (no time component) unless stated otherwise. "Today" means
the refresh's run date, London time.

## 1. Scope and counting

**In-scope epics** are the child epics of the PCW (`parent = PCW-XXXX`) that the owner chose to
track at setup, minus any added to the PCW later and not yet included (see §4 Drift).

**Counted tasks** are issues of the confirmed counted types (default: Task, Story, Bug - confirmed
per-tracker at setup) whose `parent` is an in-scope epic, minus Excluded and Unmapped (see
buckets, §2). Sub-tasks never count, regardless of type. Epics never count as tasks, even if an
epic's own type is in the counted-types list.

- Fetch children of an epic with `parent = <epic key>` in JQL, not by reading `parent` back off
  an issue (the connector drops that field on read - see SKILL.md site facts).
- A type seen on a child that isn't in the counted-types list and isn't explicitly excluded
  (Sub-task, the testing-lane types) is unexpected: surface it to the owner rather than silently
  including or dropping it.

**Progress is binary, pooled, equal-weight.**

- A task counts as Done only when its status maps to the Done bucket. In PR and In QA get no
  partial credit - no 50%, no fractional credit of any kind.
- Headline % complete = (counted Done tasks across all in-scope epics) / (all counted tasks
  across all in-scope epics). This is a pooled ratio, never an average of per-epic percentages.
- Each epic also shows its own % complete, computed the same way scoped to that epic's tasks.
- Every task weighs 1. Config carries `unit: "task"` so points-weighting could be added later
  without changing this spec's shape - but version 1 never reads story points.
- Tasks left = counted tasks not in the Done bucket, Blocked included.

## 2. Status buckets

Eight buckets: Not started, In development, In PR, In QA, Done, Blocked, Excluded, Unmapped.

- The owner maps every observed status (on tasks, and separately on epics) to one of these at
  setup, via `references/walkthrough.md` §3. The mapping is versioned in config
  (`workItems.statusMapping.version`); bump the version on any change, bump
  `workItems.statusMapping` current snapshot too.
- **Excluded** (e.g. Won't do, Cancelled): the task leaves the denominator entirely - it neither
  lifts nor drags % complete. Still shown as a count on the headline pills.
- **Unmapped**: a status seen on a counted-type task that isn't in the mapping (new status, or a
  status that appeared after setup). Shown as a grey bucket with the status name(s) listed.
  Excluded from % complete and from run rate until the owner maps it via `/pcw-tracker scope`.
- Headline pills show the first six buckets (Not started, In development, In PR, In QA, Done,
  Blocked) with counts; Excluded and Unmapped show as small counts alongside, not blended in.
- **Epic status** (for "is this epic still open") is a separate, simpler judgement: an epic is
  open unless its own status is Done, Won't do or Cancelled. This uses the epic's own status
  value directly, not the task bucket mapping.

## 3. Testing lane (Phase 2 - not computed in Phase 1)

Testing Epics and their children (Test Run, Test Case/Test Cases, Exploratory Testing, Task) are
never blended into the headline in any phase. In Phase 1 they appear only under Not Tracked with
reason "Testing". Phase 2 gives them their own lane and progress; don't build that early.

## 4. Scope transparency and drift

- Header always shows "Tracking N of M child epics" - N = in-scope count, M = total child epics
  of the PCW found via `parent = PCW-XXXX` at refresh time (M can grow - see drift below).
- **Not Tracked** list: every untracked child epic, its current status, and a reason - one of
  "Testing" (Testing Epic type), "Won't do"/"Cancelled" (epic status), or a free-text reason the
  owner gave at setup for anything else excluded.
- **Amber "untracked epics still open" notice**: count of Not Tracked epics whose own status is
  not Done/Won't do/Cancelled, split by reason, e.g. "3 untracked epics still open, 1 of them
  Testing". Clears (no notice) when that count is 0. Never let the headline show 100% while this
  notice is non-empty and non-zero outside Testing-only reasons - i.e. don't let open Testing
  work read as "missing scope", but do flag open non-Testing untracked work prominently.
- **Drift at each refresh**, compare this refresh's `parent = PCW-XXXX` result against config's
  in-scope list and against last refresh's `current`:
  - A counted task in an unmapped status → grey Unmapped bucket (§2), not blocking the refresh.
  - A new child epic on the PCW not in config's in-scope or excluded lists → listed under Not
    Tracked as "New since setup" with an amber notice; never auto-included. Needs `/pcw-tracker
    scope` to bring it in.
  - A new task appearing under an already in-scope epic → counted automatically; recorded in
    that day's snapshot as scope added (§7).
- Each snapshot records tasks added, removed (no longer returned by the epic's child query - e.g.
  moved out, deleted) and excluded since the previous snapshot, so a moving total is explained,
  never just a bare delta.

## 5. Dates, working week, target dates

**Working week**: Monday–Friday. Holidays: an optional per-tracker list in config, default none.
Working weeks left (from date A to date B inclusive) = working days between them / 5.

**Target date per epic**: epic-level commitment (Phase 2 - not in v1) if one exists, else the
end-date field chosen at setup (one of Planned End Date / Est. Due Date / Due date - confirmed
per-tracker), else none. No date → epic renders as a grey "No date" row, excluded from pace maths
entirely (not zero, not blank - absent from the calculation).

**Headline target date**, in order, first that exists wins:
1. PCW-level commitment (§8)
2. Latest due date among in-scope epics (using the chosen end-date field)
3. PCW's own Planned End Date (`customfield_11096`)
4. Unknown - render "Unknown", no pace maths against it.

If the resolved headline target date has passed and tasks remain, show "N working days over"
instead of a rate (N = working days from that date to today).

## 6. Run rate, required rate, signals, forecast

**Run rate** (per epic, and pooled for the headline) = (tasks Done now) − (tasks Done 7 calendar
days ago), read from the Jira changelog's status-field transitions into the Done-mapped status,
expressed as "per week". On day one (no prior snapshot 7 days back to compare against), compute
this directly from the last 7 days of changelog history - a live calculation, not a stored
snapshot row, and never backfilled into `snapshots/*`.

**Required rate** = tasks left / working weeks left (to the relevant target date - epic's own for
an epic signal, headline target date for the headline signal).

**Epic signal** (and the headline signal when no epic has a date - see below):

- Green: required ≤ actual, OR all tasks done, OR the epic/window's start date is in the future.
- Amber: required ≤ 1.5 × actual (the multiplier is a per-tracker parameter,
  `rules.ragMultiplier`, default 1.5).
- Red: required > 1.5 × actual, OR the target date has passed with tasks left, OR the epic
  conflicts with a commitment (§8).
- **Zero-actual fallback**: when actual (run rate) is 0, signal by absolute thresholds on
  required rate instead (parameters `rules.fallbackThresholds`, default green < 3/wk, amber
  3–4/wk, red > 4/wk).
- Every signal pill renders its numbers inline, e.g. "needs 1.3/wk, doing 24.0/wk"-never a bare
  colour.

**Headline signal** = the worst (reddest) signal among in-scope epics that have a target date. If
no in-scope epic has a date, apply the same signal rules to the pooled counts against the
headline target date (§5). If there is no date anywhere (headline target date is Unknown), show
Unknown - no colour, no fabricated signal.

**Forecast finish** = today + (tasks left / run rate) expressed in working weeks, pooled
(headline) or per-epic. Shown against the relevant target date as "N working days early/late".
Run rate = 0 → "No forecast" (never a fabricated infinite date). Label every forecast: "indicative
- assumes the current 7-day pace continues, each task counts equally."

**Early pace caveat.** Compute working days of real history = working days from the earliest
in-scope epic's start date (or the reporting start date if none has one) to today. Under 7 working
days, run rate and forecast are based on less than a full trailing week and read noisier than
they'll be once more data accumulates. Show the "Early data - only N working days of activity so
far" note next to run rate regardless. **Forecast finish itself is suppressed while pace
confidence is low** - show "Forecast finish: available once there's a full week of run-rate
history (N working days so far)" instead of the date. Updated after PCW-1311's build: a first pass
only caveated the number ("indicative, provisional"), but a 2/week rate from 3 days of history
still read as a flat, alarming 101-days-late forecast even with the caveat attached - Deji asked
for it to not appear at all until there's enough pace data behind it. The number reappears
automatically once `paceConfidence.low` turns false (7+ working days of history); nothing else
needs to change for that transition.

**Time elapsed vs work done: dropped.** Was in the original brief; removed after the first live
build showed it alongside the headline % (which already carries "work done") - Deji's call: "adds
no value + work done" (the duplicate was the problem, not just the elapsed half). Do not recompute
or display `windowStart`/`windowEnd`/elapsed percent anywhere on the page.

**Epics with no counted tasks** render "No tasks yet" (grey), are excluded from all pooled maths,
and count toward the data confidence strip (§9).

## 7. Snapshots and reporting start

- Reporting starts at build time, or at a later date the owner chose at setup - never earlier,
  never backfilled. The PCW's prior history before that date is not reconstructed.
- First snapshot, taken on the reporting start date, shows the true current state - a PCW that's
  5% done opens at 5%, not 0%. % complete always covers all in-scope tasks regardless of when
  they were completed.
- If the owner chose a later start: build and config happen now; `snapshots/*` and the scheduled
  refresh begin writing from that date. Until then the page shows "Reporting starts on [date]"
  instead of a snapshot table.
- One snapshot per working day, keyed `snapshots/<YYYY-MM-DD>`, written only by the owner's
  refresh - never by a viewer opening the page. Refreshing twice in one day replaces that day's
  document (same key, overwritten), not a second row. A working day with no refresh is simply
  absent - the page shows "no snapshot" for that date rather than interpolating.
- Each snapshot stores what it was computed from: the scope (epic key list), the status-mapping
  version, the commitments in force (with their dates), the epic due dates seen, and the skill
  version. Render a marker on any row where config changed since the previous row (e.g. "mapping
  changed 14 Oct") so a jump in % isn't misread as delivery progress. Flag any epic due date that
  moved since the previous snapshot ("ABC-123 due date moved 14 Nov → 28 Nov").
- Snapshot table on the page shows the last 30 working days; older rows collapsed (not deleted -
  `snapshots/*` keeps everything; the page just limits what it displays by default).
- Run rate, forecast, days-in-current-status and the no-progress rule read the Jira changelog
  directly (including history from before the reporting start where needed for a 7-day window);
  none of that reaches back into `snapshots/*` and none of it is backfilled as snapshot rows.

## 8. Commitments

- A commitment is a contractual or non-negotiable date, entered only at setup or via an explicit
  owner edit - never inferred. PCW-level by default (one commitment covers the whole tracked
  scope). Epic-level commitments are Phase 2.
- Fields: date, label (free text, e.g. "Contractual delivery to AMC"), optional reference
  (contract/SOW string).
- Shown as a hard line on the Gantt (Phase 2) and, in Phase 1, as "N working days remaining" in
  the header, separate from the PCW's own Planned End Date.
- **Conflict, flagged red** when: an in-scope epic's due date is after the commitment date, OR
  (Phase 2) an epic-level commitment is later than the PCW-level one, OR the PCW's own Planned End
  Date is after the commitment.
- **Owner-only edits, with a trail.** Every change stores `{oldDate, newDate, changedAt, reason}`.
  The page renders the trail, e.g. "Commitment moved 28 Aug → 15 Sept, changed 3 Oct: extension
  agreed."

## 9. Needs attention

Rule-based, no generated prose - every line is a template filled from computed numbers, e.g.
"ABC-123: needs 3.2/wk, doing 1.1/wk, due 14 Nov." Show the top five by rule order below, plus
"+n more" if more trip. If nothing trips, show "Nothing needs attention" - never an empty panel
with no explanation.

Rule order (first-tripped-first-shown within each rule, then next rule):

1. Commitment conflicts (§8) and Breached dependencies (§10)
2. Red epics, and epics past their target date with tasks left
3. At Risk dependencies (§10)
4. Blocked tasks - count, and the single longest-blocked task's days blocked
5. No net progress for N working days (parameter `rules.noProgressDays`, default 3) - net Done
   count unchanged across N consecutive working days
6. Stale tasks (§11), longest-idle first
7. Epic due-date moves in the last 7 days, and scope added ≥ X% of counted tasks in the last 7
   days (parameter `rules.scopeGrowthThreshold`, default 10%)
8. Epics with no dates or no tasks
9. Unmapped statuses, and new epics since setup
10. Data older than one working day (staleness of the snapshot itself, not of tasks)

## 10. Dependencies

- Source: Jira issue links only (type chosen at setup; default Blocks / "is blocked by"). Never
  inferred from epic numbering or order.
- Covers links between two in-scope epics, plus a link from an in-scope epic or the PCW itself to
  anything outside scope (shown as External). Task-level links are out of v1.
- States:
  - **Clear**: the blocking issue is Done.
  - **At Risk**: blocker not Done, AND (the blocked epic has already started OR the blocker's due
    date is later than the blocked epic's due date).
  - **Breached**: blocker not Done AND past its own due date.
  - **Unknown**: blocker not Done and there are no dates on either side to judge by.
- No links found → render "No dependency links recorded in Jira." Never render "no risk" - the
  absence of recorded links is not the same claim as the absence of risk.

## 11. Stale work

- Stale = an active task (status bucket In development, In PR or In QA) with no Jira update for
  more than N working days (parameter `rules.staleDays`, default 1; optional override per bucket
  in config). Computed as of the refresh time; label the figure with that timestamp.
- Blocked tasks are excluded from "stale" (they have their own bucket and their own "days
  blocked" figure) - don't double-surface a blocked task as also stale.
- "Days in current status" (read from the changelog's last transition into the current status) is
  shown on every active task as the safety net, because Jira's own `updated` timestamp moves on
  any edit (a comment, a field tweak), not just a status change.
- Sort stale lists by days idle, longest first.

## 12. Reconciliation

At build and at every refresh: for each in-scope epic, compare the counted-task total computed
from the cached/fetched issue list against an independent JQL count
(`parent = <epic> AND issuetype in (<counted types>) AND status not in (<excluded statuses>)` or
equivalent). Any mismatch is flagged on the page and surfaced in Needs attention - never silently
reconciled by trusting one number over the other.

## 13. Data confidence strip

Shows: % of in-scope epics that have a usable date, count of epics with no tasks, Unmapped bucket
count, last refresh timestamp, and "Reconciled with Jira at [time]" (from §12).

## 14. Page staleness warning

The page always shows "Data as of [date, time]" from the stored `current` document. If that
timestamp is older than one working day at the time the page is viewed, show a visible warning.
Reload re-reads stored data; it never calls Jira directly (see `references/page-template.md`).
