# v222 — build list

**Status: OPEN — collecting items. Not built.** Per Mario's standing rule, nothing gets built
until he says go. Base file: v221 (on the project shelf; its Pages deploy was stuck after
GitHub's 2026-10-05 outage — confirm v221 is actually live before building on it).

---

## 1. Disposed cases stop producing court dates — APPROVED 2026-10-08

**Found on:** Taelor May — case 2026PF29233 (County Court 7) recorded as Dismissed, client
status Resolved (still collecting), yet an Oct 8 Arraignment (linked to that case, "No court
set") still showed as the card's Next Court Date with a "Today" pill, and on the calendar.

**Root cause (verified in v221):** nothing that reads `c.events` checks case or client
status. `renderCalendar` (grid and upcoming list) loops every event from every client.
The only filter anywhere is `ev.done`.

### 1a. Shared helper — `_eventIsMoot(ev, c)`
Returns true when the event should be suppressed:
- `ev.caseId` points to a case whose `caseStatus` is `dismissed`, `resolved` or `closed`; **or**
- the event has no `caseId` (older entries) **and** client `status` is `resolved` or `closed`
  (Mario's default, 2026-10-08 — approved).

**Never moot:** `probation`, `jail`, `pending`, `active`, or any status not listed.
Probation reviews, compliance hearings and MTR settings are real court dates.
**Deny-list on purpose** — the opposite of v200's `NAG_STATUSES` allow-list — because the
failure mode here is a real court date disappearing. An unknown/new status must *show*.

Past events stay visible as history; the helper only matters for today-and-forward.

### 1b. Apply the helper at every consumer
- `renderCalendar` — grid chips and the upcoming list below it
- `calDayClick` / day list
- `renderDashCal`, `_dashCalItems`, `getUpcomingEvents`
- `checkCourtReminders` (notifications)
- `_checkUpcomingCourtBannerInner` — **highest stakes**: the 48-hour banner offers to text or
  email the client. A dismissed client must never get a "dress for court" reminder.
- `buildDailySummaryEmail`
- `_nextEventRowHtml` (client card headline — see item 2)
- Leave the EOD report alone (today's events only, historical record).

**Worker check needed:** confirm whether `/send-briefing` builds its own court list from
Firebase or only sends the HTML the app hands it. If it builds its own, the same rule has to
go into the Worker or the briefing will still list moot dates.

### 1c. Clean-up prompt on disposition
When `saveResolution()` (or Edit Case) moves a case to Dismissed / Resolved / Closed and that
case still has future undone events: confirm dialog —
"This case still has N future court date(s). Mark them done?"
Yes → set `done:true` (do not delete — keeps the record). No → leave them; 1a hides them anyway.
This fixes the data, not just the display.

---

## 2. Client card — resolved-but-collecting state — APPROVED 2026-10-08 (mockup shown)

### 2a. Status badge
- Client `status==='resolved'` and `balanceDue(c)>0` → amber badge **"Resolved · collecting"**.
- `status==='resolved'` and `balanceDue(c)<=0` → **"Resolved · paid"**, plus a small prompt to
  move the file to Closed.
- Other statuses unchanged.

### 2b. Headline row
When **every** case on the file has `caseStatus` in dismissed / resolved / closed, the
"Next Court Date" row is replaced by a green **Disposition** row:
- ✅ `<Resolution type> · <case no.>` (e.g. "Dismissed · 2026PF29233")
- Second line: `<court> · no court dates pending`
- Right side: **"Balance due"** chip when `balanceDue(c)>0` — **no dollar amount**, because
  the card's charged/paid/due figures sit behind the lock box.
- Multiple disposed cases → most recent disposition, with "+N more" if needed.

If any case is still live (including probation/jail), the normal Next Court Date row shows.

---

## 3. Carried over — lead view modal fee block
Confirmed build item from 2026-10-06: the lead view modal doesn't show the fee quote /
down payment block when those fields are empty. Needs a fix.

---

## Conventions for this batch
New number + filename + bumped `<title>` · `node --check` every inline script block · no
nested template literals · complete file for copy-paste replacement · v221 kept as rollback ·
full GitHub Pages URL with the build: https://thejesus4.github.io/gamezlaw/GamezLaw_v222.html
