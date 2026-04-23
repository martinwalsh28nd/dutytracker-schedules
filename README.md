# DutyTracker — schedule data

Public data files the DutyTracker iOS app reads at runtime.

Currently hosted here:

- **`lgh/call-schedule.json`** — Lutheran General Hospital, General
  Surgery call schedule. Source: scheduling coordinator's emailed PDFs.
  Updated manually when a new PDF arrives (or automatically once the
  email → JSON pipeline lands via the Content Pipeline thread).

Not hosted here (yet):

- **Christ / CMC call schedule** — still lives in the program's public
  Google Sheet at `https://docs.google.com/spreadsheets/d/1U5t-QabreXeXQwoK2apv-l6qv6dfJcgB1wnbjDOTPpQ`.
  The app fetches CMC directly from that sheet. If the CMC source ever
  moves, it'll land here too for symmetry.

## How the app reads this

The iOS app fetches the raw-content URL:

```
https://raw.githubusercontent.com/martinwalsh28nd/dutytracker-schedules/main/lgh/call-schedule.json
```

`raw.githubusercontent.com` has a short-TTL cache (~5 min), so a commit
to `main` shows up to the app within a few minutes of push.

## File shape — `lgh/call-schedule.json`

```jsonc
{
  "hospital": "LGH",
  "hospital_name": "Lutheran General Hospital",
  "program": "General Surgery",
  "updated_at": "YYYY-MM-DD",       // date the JSON was last updated
  "source": "free-text provenance note",
  "call_definitions": { /* see file */ },
  "schedule": [
    {
      "date": "2026-04-22",           // ISO date (yyyy-MM-dd)
      "dow": "Wed",
      "intern":       "White",        // Row 1 under the date — intern call
      "intermediate": "Gokhale",      // Row 2 — night float / intermediate
      "chief":        "Smith",        // Row 3 — chief (may be cross-program)
      "vacations":    ["Sun", "Mckay"] // Row 4 — residents out that day
    }
    // ... one entry per day the schedule covers
  ]
}
```

Any day not present in `schedule[]` means "not in the currently
published schedule" — the app should render that as unknown, not empty.

## Row meanings (per the scheduling coordinator's PDF)

Under each date there are four rows of names:

- **Row 1 — `intern`**: intern call. Takes floor calls from General
  Surgery teaching services, Thoracic, Vascular, and Trauma.
- **Row 2 — `intermediate`**: intermediate / night-float call.
  Equivalent to CMC's night-float role. Takes consults to the ER, ICU,
  SICC, RR, and gets notified of every ER admission.
- **Row 3 — `chief`**: chief call. Senior backup for intern and
  intermediate. Chiefs rotate in from other programs — the name on the
  schedule is authoritative even if it's not on the published Advocate
  General Surgery residents list.
- **Row 4 — `vacations`**: residents on vacation that day.

## Call hours (LGH-specific)

Different from CMC:

- **Monday–Thursday**: 1700 → 0600 next day
- **Friday**: 1700 Fri → 0700 Sat
- **Saturday**: 1000 Sat → 0700 Sun
- **Sunday**: 1000 Sun → 0600 Mon

These hours apply to the on-call window only — separate from whatever
normal service / rounding hours the resident owes during the day.

## Updating the LGH schedule manually

When a new PDF arrives by email:

1. Edit `lgh/call-schedule.json`
2. Bump `updated_at` to today
3. Update `source` with the new email date
4. Replace or append entries in `schedule[]`
5. `git commit && git push` to `main`
6. The app picks up the change within a few minutes (raw GH CDN)

If you'd rather send me the PDF and have me regenerate, drop the file
in the DutyTracker coordination thread and I'll do it.

## Automated flow (future)

A Claude-scheduled agent in the **Content Pipeline thread** will:
1. Poll Gmail every ~4 hours for the Lutheran scheduling email
2. Download the PDF / DOCX attachment
3. Parse call assignments out of the table
4. Write the parsed schedule back into `lgh/call-schedule.json` in this repo
5. Commit + push

When that agent lands, human manual edits still work — commits on top
of the bot's output will be respected as the latest truth.
