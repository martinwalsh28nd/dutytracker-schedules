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
      "date": "2026-04-22",          // ISO date (yyyy-MM-dd)
      "dow": "Wed",
      "first_call":  "White",        // 1ST call, usually PGY-2
      "second_call": "Gokhale",      // 2ND call
      "snr_call":    "Smith",        // SNR backup for 1ST + 2ND
      "students":    ["Sun", "Mckay"] // STU-A daytime / STU-B overnight
    },
    // ... one entry per day the schedule covers
  ]
}
```

Any day not present in `schedule[]` means "not in the currently
published schedule" — the app should render that as unknown, not empty.

Call-time definitions (pulled from the PDF footer):

- **1ST call**: All floor calls from General Surgery teaching services,
  Thoracic, Vascular, and Trauma Services.
- **2ND call**: Consult to ER, ICU, SICC, and RR. 2ND gets notified of
  all ER admissions.
- **SNR call**: Consult backup for 1ST and 2ND.
- **Weekday call**: 1700 → 0600 next day (0700 if it's a Friday call).
- **Weekend call**: 0700 → 0700 next day (0700 if it's a Sunday call).
- **STU-A** (student day): 0600–1800. Rounds with team, then follows
  the intern or 2-year on call.
- **STU-B** (student night): 1800–0600 overnight. Follows the intern or
  2-year on call.

## Updating the LGH schedule manually

When a new PDF arrives by email:

1. Edit `lgh/call-schedule.json`
2. Bump `updated_at` to today
3. Update `source` with the new date if relevant
4. Replace or append entries in `schedule[]` (see file shape above)
5. `git commit && git push` to `main`
6. The app picks up the change within a few minutes (raw GH CDN)

If you'd rather send me the PDF and have me regenerate, that works too
— drop the file and tag me in the DutyTracker coordination thread.

## Automated flow (future)

A Claude-scheduled agent in the **Content Pipeline thread** will:
1. Poll Gmail every ~4 hours for the Lutheran scheduling email
2. Download the PDF / DOCX attachment
3. Parse resident-call assignments out of the table
4. Write the parsed schedule back into `lgh/call-schedule.json` in this repo
5. Commit + push

That agent hasn't been built yet. When it lands, human manual edits
still work — commit on top, the agent will respect whatever's on
`main` as the latest truth.
