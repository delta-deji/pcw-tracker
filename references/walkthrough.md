# First-time walkthrough

Runs once, on `build PCW-XXXX` with no stored config. Can be paused and resumed - save progress
to a draft config as each answer lands, don't hold it all in conversation state only. Nothing is
published, shared, scheduled or written to Jira until the final confirmation (§9).

Ask the essential questions (§2–6) **one at a time** - each answer shapes the next. Pre-fill every
answer you can from Jira so Deji only has to correct, not compose from scratch. Outside this
walkthrough, the normal rule applies (all feedback at once) - this is the deliberate exception.

## 0. Opening explainer (five lines, before any question)

Say, adapted to the real PCW key and title once discovered:

> I'll set up a delivery tracker for [PCW-XXXX: title]. It measures task-count progress, pace
> against deadlines, blockers, stale work, dependencies and scope change - all traced to Jira.
> It does not measure quality, effort, capacity, budget or confidence. I'll ask five short
> questions, pre-filled from Jira where I can; you only need to correct what's wrong. Nothing
> gets built, published or scheduled until you confirm at the end.

## 1. Read the PCW, list child epics

- `getAccessibleAtlassianResources` once if cloudId isn't already known this session.
- `getJiraIssue` on the PCW key, compact or evidence view - get summary, status, the date fields
  named in SKILL.md, and labels.
- `parent = PCW-XXXX` in JQL to list child epics: key, title, project, issuetype, status. Request
  date fields explicitly (empty ones come back as stubs - see SKILL.md site facts).
- **For each child epic, also fetch `getJiraIssue(key, view:"full")` to read its standard `duedate`.**
  JQL search drops this field no matter what view or fields list is passed - it only ever comes
  back through a direct per-issue full-view read. Skipping this step is exactly how PCW-1311's
  build twice concluded "no epic has a due date" when three of them did. Do this before telling
  Deji which epics have dates - a JQL-only pass will under-report.
- If the PCW itself doesn't exist or the key is wrong, stop and say so - don't guess a near match.

## 2. Scope

Ask: **"Which of these are you tracking delivery of?"** - show the full child-epic list from §1
with a pre-filled default: everything **except** Testing Epics (issuetype) and epics whose own
status is Won't do or Cancelled. Record a short reason for every excluded epic (used by
`config.scope.excludedEpics` and the Not Tracked list). Offer Testing Epics explicitly as "leave
out" or "Testing lane in Phase 2" - Phase 2 isn't built yet, so for now they're always left out,
just with reason "Testing" instead of a generic exclusion reason.

## 3. Status mapping

- List every status found on **counted-type** tasks across the in-scope epics, with counts, plus
  the statuses found on the epics themselves (separately - epics use a simpler open/closed
  judgement, not the task bucket mapping).
- Pre-suggest obvious mappings: anything that reads as a backlog state → Not started; Blocked →
  Blocked; Won't do / Cancelled → Excluded; Done → Done.
- Ask explicitly about anything ambiguous (e.g. "Ready for Release" could be Done or In QA -
  don't guess which).
- Also confirm the **counted issue types** actually present (default Task, Story, Bug) - if the
  epics contain other types on top of the brief's "seen" list (Sub-task, Test Run, Test Case(s),
  Exploratory Testing), confirm Sub-task is excluded and testing types are out of scope for v1,
  per `references/compute-spec.md` §1 and §3.

## 4. Reporting start

Ask: **"Report from today, or from a later date?"** Default today. Never earlier than today -
say so if Deji asks for an earlier date, don't silently allow it.

## 5. Commitment

Ask: **"Is there a contractual deadline or commitment you're tracking towards?"**
- Offer the PCW's own Planned End Date and Handover Date (if populated) as candidates, worded as
  candidates, not defaults - never assume either is the commitment without being told.
- If yes: date, a label (e.g. "Contractual delivery to AMC"), optional reference (contract/SOW).
- If no: leave `commitments.pcwLevel` unset; the headline target date falls back through the
  chain in `references/compute-spec.md` §5.

## 6. Share and schedule

- **Who can view**: named people (list emails/accounts) or whole organisation - internal only,
  never public, never clients. Record as `operations.viewers`.
- **Daily refresh time**: default 08:45 UK time, working days only. Record as
  `operations.refreshTimeLocal` / `refreshTimezone`.
- Make clear at this point: the schedule itself isn't created yet - it's created after the final
  confirmation (§9), live, because this environment requires an explicit approval moment to
  create a scheduled task (see SKILL.md "Scheduling"). If that approval fails for any reason, say
  so plainly and fall back to manual refresh - don't silently redesign around it.

## 7. Defaults - shown, not asked

List, don't interrogate: counted issue types (§3), date fields used (start/end - ask which pair if
Deji hasn't implicitly chosen one via §5, otherwise default to Planned Start/Planned End), working
week (Mon–Fri, fixed), stale threshold (1 working day), RAG multiplier (1.5) and fallback
thresholds (green <3/wk, amber 3–4/wk, red >4/wk), dependency link types (Blocks).

Ask once: **"Change any of these? (default no)."** Anything advanced stays on its default unless
Deji says yes.

## 8. Confirm

Show a plain-language summary: what's tracked (N of M epics), from when, against which date(s),
who can see it. **Build, publish and schedule nothing until this is confirmed.** Deji's
confirmation here also authorises the scheduled refresh to write to the tracker's own storage
each day - say that explicitly, don't bury it.

## 9. After confirmation

1. Write `config` (first write - no `if_version`).
2. Run the compute pipeline (`references/compute-spec.md`) for today, producing `current` and the
   first `snapshots/<reportingStart>` (or "Reporting starts on [date]" placeholder if a later
   start was chosen).
3. Publish the page starting from **`templates/page.html`** (the approved base design, not a
   from-scratch build) - copy it, change the `<title>` tag and nothing else structurally; it reads
   everything else live from `db`, so a fresh PCW needs no other edits to the HTML. Declare `db` +
   `user` capabilities, rules scoped so only the owner (and co-owner, if set) writes. Review
   `references/page-template.md` for the rules behind that design before changing it for any
   reason - don't quietly redesign per-PCW.
4. Share it per §6 - this is the moment that needs Deji to act on the artifact's own Share menu;
   tell him exactly what to click, don't attempt it via a tool call.
5. Create the scheduled refresh live, with Deji present to approve it. On failure, report it and
   fall back to manual refresh - don't retry silently or pick a different scheduling mechanism
   without saying so.
6. Ask the end-of-session defaults question (§10).

## 10. End-of-session defaults question

At the end of **any** setup or scope-change session, ask: **"Is there anything here that should
become a default in the skill?"** If yes, propose the specific edit to `SKILL.md` /
`references/compute-spec.md` defaults and to `CHANGELOG.md`, and wait for approval before writing
it - this changes the skill for every future tracker, not just this PCW's config.

## `/pcw-tracker scope PCW-XXXX` (not first-time)

Read the stored config. Re-run §1's discovery, diff against config's `scope.inScopeEpics` /
`excludedEpics` and `workItems.statusMapping`. Ask only about what's new or changed - new child
epics, new statuses - not the whole walkthrough again. Update config with approval, append to any
history trail affected (commitments, mapping version), recompute.

If discovery finds nothing new or changed, say so plainly rather than silently doing nothing - a
clean "nothing's drifted" result is itself useful information, not a non-event. It's also fair to
proactively revisit one existing exclusion whose original reason may no longer hold (e.g. a
"placeholder" epic that's since gained real dates and activity) - ask about it directly rather than
assume the first-setup reason still stands forever. Whatever the owner decides - keep it excluded,
or bring it into scope - record the decision: append a `reconfirmations` entry (see
`references/config-schema.md`) if kept excluded with the same reason, or move it to
`inScopeEpics` and recompute if brought in. Either outcome is a real config write, not a no-op.
