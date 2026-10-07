# Page template

The published page is one self-contained HTML artifact, published once (and republished only on
an `update`, a scope change needing a layout change, or a skill upgrade - never on a daily
refresh). The daily refresh writes only to the shared `db` (`current`, `snapshots/<date>`); the
page reads that data live. **Capabilities: `db` + `user` only - never `artifact`.** The `artifact`
capability is for pages that republish their own HTML on viewer interaction; this page's content
changes because Jira changed, not because a viewer did something, so it belongs entirely to `db`.

## Capability wiring

```js
capabilities: {
  db: {
    rules: [
      { path: "", read: "view", write: "owner" },
      // config/current/snapshots are owner-written, everyone-admitted reads them (default root
      // read level is already "view" - listed explicitly here for clarity, not because it
      // changes the default)
    ]
  },
  user: {}
}
```

- Default write level is `interact` (Contributor) and up - too loose for this tracker, where only
  the owner (and optional co-owner, both tracked by accountId in `config.owner` /
  `config.coOwner`) should ever write. The explicit `write: "owner"` rule locks that down; the
  refresh logic itself must also check the acting account against `config.owner`/`coOwner` before
  writing, since the `db` rule alone governs the platform's enforcement, not the skill's.
- `user` is declared so the page can call `user.isOwner()` / `user.canEdit()` to decide whether to
  show any owner-only affordance (there shouldn't be many - this is a read-only page for everyone
  except the refresh process itself).
- No `assets`, `mcp`, `room`, `sample`, `comments`, `artifact` or `files` - none of those apply.

## Reading data in the page

```js
const db = await claude.use("db");
if (!db) { /* render a "sign in" or "not available" state */ }
const current = await db.doc("current/current").get();  // headline + epics + tasks (compact)
const snaps = await db.collection("snapshots").orderBy("__name__", "desc").limit(30).get();
```

**Path grammar gotcha:** `db.doc()` requires an EVEN number of slash-separated segments - a bare
`"config"` or `"current"` is one segment (a collection), not a document, and throws a `TypeError`
synchronously. Store these singletons as `collection: "config", doc_id: "config"` (path
`config/config`) and `collection: "current", doc_id: "current"` (path `current/current`) from
`ArtifactData`, and read them the same way from the page. `snapshots/<date>` is already a valid
2-segment document path, so that one needs no doubling.

Subscribe once (`onSnapshot`) rather than polling, per the `db` capability's own rules - a viewer
who leaves the page open sees the next day's refresh land live, which is a nice side effect, not
something to build extra machinery for.

## Section order (decisions first, per the brief)

1. **Header** - the `<h1>` is the PCW's own title (`config.pcwTitle`) + " Delivery Tracker", e.g.
   "Extras - Bonus Content Delivery Tracker" - not the PCW key. Directly underneath: the PCW key
   and scope line on one line, "`<key>` | N of M epics tracked". "Data as of [date, time]" with a
   visible warning if `current.computedAt` is older than one working day (compute-spec §14),
   commitment line in bold with days remaining (compute-spec §8). The "N untracked epics still
   open" notice (compute-spec §4) does NOT live here - see item 9, it sits with Not Tracked at the
   bottom instead, so the header reads as what IS being tracked, not a disclaimer about what isn't.
2. **Headline card** - % complete, status pills (compute-spec §2), progress bar, run rate,
   required rate, headline signal. **No "time elapsed vs work done" row** - dropped, see
   compute-spec §6. Forecast finish only renders once pace confidence is high enough (compute-spec
   §6's early pace caveat) - while low, show the "available once there's a full week of history"
   placeholder instead of a date.
3. **Needs attention** (compute-spec §9) - top 5 + "+n more", or "Nothing needs attention".
4. **Daily snapshot table** (compute-spec §7) - date, % complete, tasks done/in progress/total,
   deltas vs previous row, config-change and date-move markers. Last 30 working days, older rows
   collapsed behind a disclosure, not deleted.
5. **Epic run rates** - expandable rows: tasks left, % complete, weeks left, signal pill with its
   numbers (compute-spec §6).
6. **Gantt** - a real timeline, not a decorative one: a visible weekly date axis with gridlines
   that line up with every epic's bar, each epic's own start/due date spelled out in text beside
   its row (never rely on a hover tooltip alone - a static page viewer won't find it), a solid
   today-line and a dashed commitment-line. Epics with no date render as a plain "No date" row.
7. **Task activity** (compute-spec §11) - In PR/In QA/Ready for QA, Blocked (days blocked),
   Stale (days idle, longest first). **No "Updated last working day" group** - dropped; Stale is
   the one that matters, it's the one that forces a conversation about what's stalling.
8. **Dependency risk** (compute-spec §10) - Clear/At Risk/Breached/Unknown, grouped; "No
   dependency links recorded in Jira" if none.
9. **Not tracked** (compute-spec §4) - expandable list (collapsed by default), each untracked epic
   with status + reason, with the "N untracked epics still open" amber notice placed directly
   above this disclosure, not in the header (see item 1).
10. **Footer** - how signals are calculated (plain restatement of compute-spec §6, not the
    spec itself), what the tracker does not measure (quality, effort, capacity, budget,
    confidence), a "How to read this" link to `README.md`'s rendered content (inline, since the
    page can't link out to a repo file a viewer may not have access to - reproduce the relevant
    README sections directly in the footer's expandable "How to read this", don't hyperlink to a
    file path).

## Plain English on the page

- **Label it "Status" on the page, never "Signal".** `compute-spec.md` calls this value a "signal"
  as its own internal term (it's a useful word for a rule-based traffic-light output), but shown
  bare as a label next to "Critical" it's meaningless to a reader - Deji's words: "Signal means
  nothing!" The value itself (On track / Under pressure / Critical) is already plain English; only
  the label needed fixing. Always pair the value with its reason line right underneath (already
  computed as `signalReason`) so "Critical" is never shown without "why" in the same glance.
- **"How signals are calculated" (footer) reads as "How the status... is worked out"**, in full
  sentences anyone can follow, not a formula. Write every footer/explainer sentence this way, not
  as a glossary definition - say what's being compared to what, then what each outcome means.
- When in doubt whether a word is plain English, assume the reader is not a PM and not technical -
  Deji himself is the PM here, but the page will be read by others on the delivery team too.

## Row alignment with variable-height content

When a row pairs a multi-line block (an epic's title can wrap to 2-3 lines) with a single-line
block of numbers (tasks left, %, weeks left, a status pill), use `align-items:flex-start`, not
`center`. Centering a short block against a tall one puts the numbers at the vertical middle of
the wrapped text, which looks arbitrary rather than aligned - readers expect the numbers to line
up with the FIRST line of the title, like a list, not float independently of it. Pair this with a
shared CSS grid (`grid-template-columns`, not `flex` with ad hoc gaps) for the numeric columns, and
reuse the exact same grid on a plain-English header row above the list ("Left · Progress · % ·
Time left · Status") - a flex layout with `margin-left:auto` cannot guarantee the header's columns
land above the same pixels as the data rows' columns; a shared grid template can.

## Section headers must look like headers

Don't style `## Needs attention` etc. as muted small-caps text that blends into body copy - give
every section title real visual weight against its neighbours (e.g. full-contrast text, a short
accent-coloured underline/border, slightly larger size than body text). A section title that reads
as quietly as the content under it fails its one job.

## Visual language (from the reference screenshots - review these before writing any page code)

Reviewed `files.zip` (4 screenshots) from the brief's attachments. The actual look, confirmed:

- **Headline card**: blue left-border accent, big bold blue % (not black), status pills as
  colour-dot + label in a row above the bar, a single segmented progress bar (not a plain accent
  bar - segments coloured by bucket), then 2–3 stat boxes side by side (label in small caps grey,
  big bold number, one line of small grey context underneath). **All six headline buckets belong
  in that pill row** (Not started, In development, In PR, In QA, Done, Blocked) - caught Blocked
  missing from the row on PCW-1311's own build despite the checklist marking this item passed;
  verify the full bucket list against compute-spec §2 by eye, not by assuming a visually-plausible
  row is complete. Excluded and Unmapped fold into the quiet "Task completion - N tasks across M
  epics" label line above the pills, as ", N excluded" / ", N unmapped" - and only when non-zero.
  A first pass gave them their own pill-styled span next to the colourful bucket pills; Deji's
  call: "these create noise." A zero-value count isn't news, and a styled pill competes with the
  real pills for attention - a quiet trailing clause on the already-muted label line is enough.
- **Epic keys are links to the real Jira issue** (`https://<site>/browse/<key>`, opened in a new
  tab) everywhere a key appears as a clickable-looking element (epic rows, Gantt labels) - never a
  bare `href="#"` placeholder. A link that looks clickable and does nothing is worse than no link.
- **Daily snapshot table**: delta badges next to each changed number, small coloured pill
  (green/red/grey) showing +N or -N vs the previous row, not a bare number.
- **Epic run rates**: collapsed by default, one summary row per epic (chevron, key, title, right-
  aligned: tasks left / mini progress bar + % / weeks left / a signal pill showing the rate
  itself, e.g. "1.3/wk" or "Done" - not a separate word and number). Expanding reveals that
  epic's own tasks.
- **Gantt**: horizontal bars per epic coloured by signal, the bar itself labelled "KEY · N%", a
  vertical today-line, a legend row of the same RAG pills underneath. Epics with no date render
  as a plain "No date" row, not a zero-width bar.
- **Task activity**: grouped sections with a bold coloured label per group ("In PR (3)", "Blocked
  (9)", "Stale (0)") - not one flat table. Each row: epic key, task key, then a small detail (a
  status pill, or "N working days blocked"). The reference video also had an "Updated last working
  day" group; Deji dropped it on this build on purpose (item 7 above) - don't re-add it.
- **Dependency risk**: one row per link - "BLOCKER → BLOCKED", a short description, then a right-
  aligned state pill. Compact, not a card per link.
- **What NOT to copy**: the reference includes an "AI Risk Assessment" prose callout. This skill's
  own guardrail bans generated prose and model calls from the published page - never add anything
  like it here, however close it looks to "matching the reference."
- **Data confidence / known gaps**: keep these out of the main flow - a thin footer line, not a
  prominent card. The reference doesn't surface this at all; it's a v1 addition of this brief's
  own, so it should read as a quiet footnote, not compete with the headline for attention.

## Rules

- Works at mobile width (375px) and up; no horizontal scroll.
- Respects light/dark via `prefers-color-scheme` and the `data-theme` override, per artifact-design.
- Colour is never the only signal - every RAG pill carries its word (Green/Amber/Red) and its
  numbers as text, not just a background colour.
- No generated prose anywhere on the page. Every sentence is a template filled from `current` /
  `snapshots` data. If a sentence can't be filled because the underlying number is unknown, render
  the unknown state explicitly ("No date", "No forecast", "Unknown") - never omit the row.
- Title is literally "PCW-XXXX Delivery Tracker" (the real key, not a placeholder).
- Never embed a real observed Jira value as sample/placeholder/demo data in the HTML itself - all
  real values come from `db` at view time, not from what was true when the page was written.

## Compact task storage (`current`)

`current` holds the headline numbers, the per-epic rows, and a compact per-task array (key,
bucket, daysInStatus, blocked flag/days, updatedWithinLastWorkingDay flag) - not full Jira issue
bodies. Keep this under 256 KiB (compute-spec is designed so this is a few hundred bytes per task
at most; re-check actual size against the cap once PCW-1311's real task count is known).
