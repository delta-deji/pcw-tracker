# Config schema

`config` is the single source of truth for one PCW's tracker. One document, written only by the
owner (via the walkthrough or `/pcw-tracker scope`), read by `refresh`/`update` to know what to
compute. Keep it under 256 KiB - it holds configuration, not task data (task data lives in
`current` and `snapshots/*`, per `references/compute-spec.md` §7).

Stored at `db.doc("config")` in the published artifact's shared database.

```jsonc
{
  "pcwKey": "PCW-1311",
  "skillVersionBuilt": "0.1.0",        // bumped by /pcw-tracker update, never by refresh
  "owner": { "accountId": "...", "displayName": "Deji Ogunkoya" },
  "coOwner": null,                      // optional second accountId who may also refresh

  "scope": {
    "inScopeEpics": [
      { "key": "VC-4031", "title": "...", "project": "VC", "issueType": "Epic" }
      // ...
    ],
    "excludedEpics": [
      { "key": "DES-1125", "title": "...", "reason": "Won't do at setup" },
      { "key": "LTC-8522", "title": "...", "reason": "Testing" },
      {
        "key": "ROKU-1967", "title": "...", "reason": "Placeholder, not real delivery scope",
        // optional: present once /pcw-tracker scope has asked about this exclusion again and
        // the owner kept it. Append-only, one entry per re-ask - never overwrite a prior entry.
        "reconfirmations": [
          { "reconfirmedAt": "2026-10-07", "via": "/pcw-tracker scope", "outcome": "kept excluded, same reason" }
        ]
      }
      // every child epic not in inScopeEpics must appear here with a reason
    ]
  },

  "workItems": {
    "countedIssueTypes": ["Task", "Story", "Bug"],
    "statusMapping": {
      "version": 1,
      "task": {
        // observed status name -> bucket
        "Backlog": "Not started",
        "Ready for Dev": "Not started",
        "In Development": "In development",
        "In PR": "In PR",
        "In QA": "In QA",
        "Ready for QA": "In QA",
        "Ready for Release": "Done",       // example only - confirm per tracker, brief flags this as ambiguous
        "Blocked": "Blocked",
        "Done": "Done",
        "Won't do": "Excluded"
      },
      "epic": {
        // separate, simpler: which epic statuses count as "closed" for Not Tracked purposes
        "closedStatuses": ["Done", "Won't do", "Cancelled"]
      }
    },
    "dependencyLinkTypes": ["Blocks"]
  },

  "dates": {
    "endDateField": "customfield_11096",   // chosen at setup: Planned End Date | Est. Due Date | duedate
    "startDateField": "customfield_11145", // chosen at setup: Planned Start Date | Start date | Est. Start Date
    "workingWeek": [1, 2, 3, 4, 5],         // ISO weekday numbers, Mon-Fri fixed for v1
    "holidays": [],                          // optional ["2026-12-25", ...]
    "reportingStart": "2026-10-07"           // never earlier than build date
  },

  "commitments": {
    "pcwLevel": {
      "date": "2026-09-14",
      "label": "Handover to AMC+",
      "reference": null,
      "history": [
        // { "oldDate": null, "newDate": "2026-09-14", "changedAt": "2026-10-07T13:00:00Z", "reason": "initial setup" }
      ]
    }
    // epicLevel: []-Phase 2
  },

  "rules": {
    "ragMultiplier": 1.5,
    "fallbackThresholds": { "greenUnder": 3, "redOver": 4 },
    "staleDays": { "default": 1 },
    "noProgressDays": 3,
    "scopeGrowthThreshold": 0.10
  },

  "operations": {
    "refreshTimeLocal": "08:45",
    "refreshTimezone": "Europe/London",
    "viewers": {
      "mode": "named",                      // "named" | "organization"
      "people": [ { "accountId": "...", "displayName": "..." } ]
    },
    "scheduleTaskId": null,                  // set once the scheduled-tasks entry is created, after explicit yes
    "lastDefaultsReviewedAt": "2026-10-07"
  },

  "page": {
    "subtitleField": null,                   // a PCW field key, or free text chosen at setup
  },

  "createdAt": "2026-10-07T13:00:00Z",
  "updatedAt": "2026-10-07T13:00:00Z"
}
```

## Notes

- `scope.excludedEpics` must account for every child epic of the PCW that isn't in
  `inScopeEpics` - this is what makes "Tracking N of M" and the Not Tracked list possible without
  re-deriving reasons from scratch on every refresh.
- `workItems.statusMapping.version` bumps on any mapping edit; `current` and each day's
  `snapshots/<date>` record the version in force when they were computed (per compute-spec §7),
  so a jump in % after a remap doesn't read as delivery.
- `commitments.pcwLevel.history` is append-only. Never overwrite a prior entry.
- `operations.viewers.mode: "organization"` means shared via general access rather than named
  invites - the config only records which was chosen, since actually granting it happens on the
  artifact's own Share menu, not via a tool call (confirmed in Phase 0).
- `operations.scheduleTaskId` stays `null` until the schedule is actually created live with Deji
  present (see SKILL.md "Scheduling"). A tracker can be fully built and manually refreshed with
  this left `null` indefinitely - that's the accepted fallback, not a broken state.
- Nothing in this document is Jira task data. If a field here ever tempts you to cache a task's
  summary or status "for convenience", stop - that belongs in `current`, not `config`.
