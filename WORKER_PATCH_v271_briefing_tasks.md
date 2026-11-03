# Worker patch — 7 AM briefing "Open Tasks" cleanup (goes with app v271)

**Deploy by hand:** Cloudflare dashboard → `gamezlaw-email` → Edit Code → make the
3 edits below → Deploy. Then test with
`https://gamezlaw-email.gamez4.workers.dev/send-briefing?force=1` — both emails
re-send immediately and the JSON response shows the task counts.

**Why this is a patch and not a whole Worker file:** the newest Worker copy I
could find anywhere (SharePoint, Drive, the repo) is **v178** (Sept 28 backup),
but the app says **v201** is deployed. Handing you a full file built on v178
would silently roll back everything from v179–v201 (the Claude office tools,
discovery flags, send_text…). These three surgical edits are safe on any
version. If you save a copy of the current Worker code to SharePoint, I can
hand you full files next time.

**What it does** — same rule as the app v271 Task boxes:

- Hides an open task tied to a **closed/resolved** client — *unless* it has a
  due date within the next 14 days (a past due date always keeps it) or it was
  added in the last 7 days. Hidden tasks are counted in a footnote, never
  deleted.
- "Tied to" = the task's `clientId` when it has one; older tasks (pre-v242)
  have none, so a full-name match against closed/resolved clients is used as a
  fallback — and if an **active** client's name also matches, the task always
  shows.
- Sorts: **overdue → due in the next 7 days → everything else** (original order).
- Caps the list at **15** with an "…and N more" line.
- Fixes a bug: the email printed `t.date` (the day the task was **added**) with
  the label "due". It now prints the real due date, red when overdue.

---

## Edit 1 — add these helpers (new code, paste once at top level)

Paste this block right **above** the line `function buildBriefingHTML(data, role, today, isWeekend) {`:

```js
/* ── v271-tasks — task noise filter (mirrors GamezLaw_v271.html) ──────────
   Hide open tasks tied to a closed/resolved client unless due within 14 days
   (any past due date keeps it) or added in the last 7 days. Overdue first,
   then due-in-7-days, then the rest in box order. Hidden = counted, never
   deleted. */
function _v271AddDays(dateStr, n) { const d = new Date(dateStr + 'T12:00:00Z'); d.setUTCDate(d.getUTCDate() + n); return d.toISOString().slice(0, 10); }
function _v271NameTokens(nm) { return String(nm || '').toLowerCase().split(/[^a-záéíóúñü]+/).filter(function(w) { return w.length >= 2; }); }
function _v271TaskMoot(t, clients) {
  if (!t) return false;
  if (t.clientId != null && t.clientId !== '') {
    const c = clients.find(function(x) { return x && String(x.id) === String(t.clientId); });
    return !!c && ['closed', 'resolved'].indexOf(String(c.status || 'active').toLowerCase()) !== -1;
  }
  const text = String(t.text || '').toLowerCase();
  let moot = false;
  for (let i = 0; i < clients.length; i++) {
    const c2 = clients[i]; if (!c2) continue;
    const toks = _v271NameTokens(c2.name); if (toks.length < 2) continue;   /* never match a one-word name */
    let all = true;
    for (let j = 0; j < toks.length; j++) { if (text.indexOf(toks[j]) === -1) { all = false; break; } }
    if (!all) continue;
    if (['closed', 'resolved'].indexOf(String(c2.status || 'active').toLowerCase()) === -1) return false;   /* an active file matches too — show it */
    moot = true;
  }
  return moot;
}
function _v271FilterTasks(list, clients, today) {
  const kept = []; let hidden = 0;
  list.forEach(function(t, i) {
    if (!t || t.status === 'done') return;
    if (_v271TaskMoot(t, clients)) {
      const keep = (t.due && t.due <= _v271AddDays(today, 14)) || (t.date && t.date >= _v271AddDays(today, -7));
      if (!keep) { hidden++; return; }
    }
    kept.push({ t: t, i: i });
  });
  function urg(t) { if (t.due && t.due < today) return 0; if (t.due && t.due <= _v271AddDays(today, 7)) return 1; return 2; }
  kept.sort(function(a, b) {
    const ka = urg(a.t), kb = urg(b.t);
    if (ka !== kb) return ka - kb;
    if (ka < 2 && String(a.t.due) !== String(b.t.due)) return String(a.t.due).localeCompare(String(b.t.due));
    return a.i - b.i;
  });
  return { tasks: kept.map(function(x) { return x.t; }), hidden: hidden };
}
```

## Edit 2 — in `sendBriefing`, replace the two task lines

Find (in `sendBriefing`, just under the sorts; the v177 comment marks it):

```js
  const attorneyTasks = asList(meta && meta.attorneyTasks).filter(function(t) { return t.status !== 'done'; });   /* v177 */
  const secretaryTasks = asList(meta && meta.secretaryTasks).filter(function(t) { return t.status !== 'done'; });
```

Replace with:

```js
  /* v271 — noise filter + ordering; `clients` is already in scope above */
  const _attyT = _v271FilterTasks(asList(meta && meta.attorneyTasks), clients, today);
  const _secT  = _v271FilterTasks(asList(meta && meta.secretaryTasks), clients, today);
  const attorneyTasks = _attyT.tasks;
  const secretaryTasks = _secT.tasks;
```

Then a few lines down, inside `const data = { ... }`, add one line after
`secretaryTasks: secretaryTasks,`:

```js
    hiddenTasks: { attorney: _attyT.hidden, secretary: _secT.hidden },   /* v271 */
```

## Edit 3 — in `buildBriefingHTML`, replace the Open Tasks section

Find (the whole ✅ Open Tasks block):

```js
  const taskList = role === 'attorney' ? data.attorneyTasks : data.secretaryTasks;
  html += '<div style="' + sectionStyle + '">';
  html += '<div style="' + headStyle + '">✅ Open Tasks</div>';
  if (taskList.length === 0) {
    html += '<div style="' + emptyStyle + '">No open tasks.</div>';
  } else {
    taskList.forEach(function(t) {
      const isWorking = t.status === 'working';
      html += '<div style="' + rowStyle + '">';
      html += (isWorking ? '⟳ ' : '⬜ ') + t.text;
      if (t.date) html += ' <span style="color:#6b7280;font-size:12px;">due ' + t.date + '</span>';
      html += '</div>';
    });
  }
  html += '</div>';
```

Replace with:

```js
  const taskList = role === 'attorney' ? data.attorneyTasks : data.secretaryTasks;
  const hiddenN = (data.hiddenTasks && data.hiddenTasks[role === 'attorney' ? 'attorney' : 'secretary']) || 0;   /* v271 */
  html += '<div style="' + sectionStyle + '">';
  html += '<div style="' + headStyle + '">✅ Open Tasks</div>';
  if (taskList.length === 0) {
    html += '<div style="' + emptyStyle + '">No open tasks.</div>';
  } else {
    taskList.slice(0, 15).forEach(function(t) {
      const isWorking = t.status === 'working';
      const overdue = t.due && t.due < today;
      html += '<div style="' + rowStyle + '">';
      html += (isWorking ? '⟳ ' : '⬜ ') + t.text;
      /* v271 — t.due is the real due date; t.date (printed before) was the day it was ADDED */
      if (t.due) html += ' <span style="' + (overdue ? badgeRed : badgeBlue) + '">due ' + t.due + (overdue ? ' — OVERDUE' : '') + '</span>';
      html += '</div>';
    });
    if (taskList.length > 15) html += '<div style="padding:8px 0;font-size:13px;color:#6b7280;">…and ' + (taskList.length - 15) + ' more in the GamezLaw Task box</div>';
  }
  if (hiddenN > 0) html += '<div style="font-size:11px;color:#9ca3af;font-style:italic;">' + hiddenN + ' task' + (hiddenN !== 1 ? 's' : '') + ' on closed/resolved files hidden — still in the app, nothing deleted</div>';
  html += '</div>';
```

---

With today's live data this changes the 7 AM emails to:

| Email | Before | After |
|---|---|---|
| Mario — Open Tasks | all 22, unsorted, "due" showing added-date | 15 shown (8 overdue first, then due-this-week), "…and 7 more", real due dates |
| Claudia — Open Tasks | all 12, unsorted | 10 shown, overdue first; footnote "2 tasks on closed/resolved files hidden" (Christina Martinez, Rodolpho Cuellar — both resolved) |
