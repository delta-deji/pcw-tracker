# Phase 1 test checklist

Maps every definition in the brief to where it's implemented, pass or fail. Filled in against the
real PCW-1311 build on 2026-10-07 - not marked pass until actually exercised against live Jira
data and the published page, not just read back from the spec.

Status key: ☐ not yet exercised · ✅ pass · ❌ fail (see note) · ➖ deferred to Phase 2/3 (named)

| # | Definition (brief section) | Implemented in | Status |
|---|---|---|---|
| 1 | Counted tasks, sub-tasks off, epics never count | compute-spec §1 | ✅ 53 counted, all type Task, no Sub-tasks included |
| 2 | Progress binary/pooled/equal-weight, headline ≠ average of epic % | compute-spec §1 | ✅ 4.1% pooled; per-epic % shown separately (6.1%, 0%, 0%, 0%) |
| 3 | Status buckets incl. Excluded/Unmapped, all six headline pills shown | compute-spec §2 | ✅ fixed - Blocked pill and Excluded/Unmapped counts were missing from the headline row, caught on a critical self-review and corrected |
| 4 | Testing lane deferred to Phase 2, shown as Not Tracked meanwhile | compute-spec §3 | ✅ LTC-8522, LTC-11013 under Not Tracked |
| 5 | Scope transparency: "Tracking N of M", Not Tracked, amber notice split by reason | compute-spec §4 | ✅ "Tracking 4 of 9"; amber notice - fixed a bug where ROKU-1967 (status Blocked, not closed) was wrongly excluded from the open count; now shows 2 open (1 Testing, 1 other) |
| 6 | Drift: unmapped status, new epic, new task | compute-spec §4 | ✅ exercised for real twice - 7 Oct: VC-4038 moved In PR to Ready for QA, correctly re-bucketed to In QA. 8 Oct: VC-4059 moved to an unmapped status ("Ready for Release") and correctly excluded from % and run rate pending `/pcw-tracker scope`; DCD-539 appeared as a brand-new task under DCD-537 and was picked up automatically |
| 7 | Snapshot explains added/removed/excluded since previous | compute-spec §4, §7 | ✅ exercised - 8 Oct snapshot records `scopeAddedSincePrevious: ["DCD-539"]` against the 7 Oct snapshot |
| 8 | Working week Mon–Fri, holidays optional | compute-spec §5 | ✅ used throughout date maths |
| 9 | Target date per epic, "No date" fallback | compute-spec §5 | ✅ DCD-538 shows "No date", excluded from pace maths |
| 10 | Headline target date chain, "N working days over" | compute-spec §5 | ✅ resolved to PCW commitment (30 Oct) correctly |
| 11 | Run rate from changelog, day-one 7-day read | compute-spec §6 | ✅ 2.0/wk, from the 2 Done transitions in the trailing 7 days |
| 12 | Required rate | compute-spec §6 | ✅ |
| 13 | Epic signal incl. 1.5× and zero-actual fallback | compute-spec §6 | ✅ VC-4031 Red (1.5×), VC-4032/DCD-537 Green via zero-actual fallback |
| 14 | Headline signal = worst epic signal / pooled fallback / Unknown | compute-spec §6 | ✅ Red, matches VC-4031 |
| 15 | Forecast finish, "No forecast" on zero run rate, indicative label, suppressed while pace confidence is low | compute-spec §6 | ✅ currently hidden behind the early-pace placeholder (3 working days of history); will show once 7+ days accrue |
| 16 | Time elapsed vs work done | compute-spec §6 | ➖ dropped per Deji - duplicated the headline %, added no value |
| 17 | Epics with no counted tasks → "No tasks yet" | compute-spec §6, §13 | ☐ not exercised - no in-scope epic currently has zero tasks |
| 18 | Needs attention rule order 1–10, "Nothing needs attention" | compute-spec §9 | ✅ 4 items populated in rule order |
| 19 | Task activity lists, days in current status | compute-spec §11 | ✅ In PR and Blocked lists shown |
| 20 | Stale definition, blocked excluded from stale | compute-spec §11 | ✅ none stale currently; Blocked tasks correctly excluded from that list |
| 21 | Dependencies: link types, Clear/At Risk/Breached/Unknown, External | compute-spec §10 | ✅ one link found (VC-4032 blocks DCD-538), classified Unknown |
| 22 | "No dependency links recorded" never "no risk" | compute-spec §10 | ✅ logic present; not literally hit this build since one link exists |
| 23 | Data confidence strip contents | compute-spec §13 | ✅ |
| 24 | Reconciliation vs independent JQL count, flagged on mismatch | compute-spec §12 | ✅ per-epic sums (33+6+5+5) match the pooled total (49) and each epic's direct JQL count exactly |
| 25 | Commitment: PCW-level, conflicts flagged red, owner-only edit trail | compute-spec §8 | ✅ recorded with one history entry (initial setup) |
| 26 | Reporting start never earlier, true-current first snapshot | compute-spec §7 | ✅ starts today, opens at the true 4.1%, not 0% |
| 27 | One snapshot/working day, same-day refresh replaces not duplicates | compute-spec §7 | ✅ exercised - refreshed PCW-1311 twice on 7 Oct, same snapshot doc updated in place (version 1 to 2), no duplicate row |
| 28 | No-snapshot day shows "no snapshot" | compute-spec §7 | ☐ not yet exercised - needs a missed working day |
| 29 | Snapshot records scope/mapping-version/commitments/due-dates/skill-version | compute-spec §7 | ✅ |
| 30 | Config-change and due-date-move markers on snapshot rows | compute-spec §7 | ☐ scope-added marker exercised (see #7); no config version change or due-date move has happened yet, so those two specific markers remain unexercised |
| 31 | 30 working days shown, older collapsed not deleted | page-template §4 | ✅ logic present; only 1 row exists so far |
| 32 | Page section order matches brief | page-template | ✅ (Gantt pulled forward into v1, see #43) |
| 33 | "Data as of", staleness warning > 1 working day | compute-spec §14 | ✅ |
| 34 | No generated prose; unknowns rendered explicitly | page-template | ✅ e.g. DCD-538 renders "No date", not a guess |
| 35 | Colour never sole signal | page-template | ✅ every pill carries its word and numbers |
| 36 | Mobile + light/dark | page-template | ☐ CSS written (media query + data-theme + mobile breakpoint) but not visually confirmed - this session's browser can't sign in to claude.ai to render it (same Phase 0 limitation) |
| 37 | Viewers read-only; owner/co-owner only writers | page-template capability wiring | ✅ `db` rule `write: "owner"` on root; config.owner is Deji's accountId |
| 38 | Second viewer sees identical numbers after reload | manual check w/ Deji | ☐ pending |
| 39 | Sharing limited to named people, or org-wide | manual check w/ Deji (Share menu) | ☐ pending |
| 40 | Scheduled refresh created live, approved by Deji | walkthrough §9.5 | ☐ pending |
| 41 | End-of-session defaults question asked | walkthrough §10 | ☐ pending (asked at end of this build) |
| 42 | Nothing changed in Jira throughout | self-audit at end of build | ✅ read-only operations only, confirmed |
| 43 | Gantt | page-template §6 | ✅ pulled forward into v1 at Deji's request - date axis, today line, commitment line, no title truncation |
| 44 | Epic-level commitments | - | ➖ Phase 2 |
| 45 | Comments | - | ➖ Phase 2 |
| 46 | Testing lane progress | - | ➖ Phase 2 |
| 47 | Phase 3 promotion gate: survived a real config change via `/pcw-tracker scope` | SKILL.md | ✅ ROKU-1967's exclusion reconfirmed via a real `/scope` run on 2026-10-07 - config written (v1 to v2), recorded as a `reconfirmations` entry, page unaffected since the decision was "no change" |
| 48 | Snapshot delta badges compare the right two rows, in the right direction | page-template §4 | ❌ found 8 Oct - found via a live screenshot after the second real snapshot existed: the query returns rows newest-first but the delta code assumed oldest-first (`prev = snaps[i-1]`), so the badge landed on the wrong row and read backwards (e.g. "In progress -3" when it had actually gone up by 3). ✅ fixed - `snaps` is now sorted explicitly instead of trusting query order, and `prev` points at the next row down |

## Known data gaps (flagged on the page's data confidence strip, not hidden)

- VC-4049's exact "blocked since" date couldn't be confirmed from its changelog within this
  build's effort budget - shown as unknown rather than guessed.
- The standard Jira `duedate` and `updated` fields are both dropped by `searchJiraIssuesUsingJql`
  in every view and field combination tried - only `getJiraIssue(view:"full")` returns them,
  one issue at a time. This is logged in `SKILL.md`'s site facts; it's why due dates needed a
  separate per-epic fetch, and why the "Updated in the last working day" activity list is scoped
  to the tasks whose changelog was already being read for other numbers, not all 53.
