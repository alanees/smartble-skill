# Duty & supervision roster (المناوبة والإشراف)

Smartble's duty module builds the daily duty/supervision rota (نموذج 8 — جدول المناوبة والإشراف اليومي): who guards the gate in the morning, who supervises the break, who stays for dismissal, plus substitutes (المناوبون البدلاء). Everything below was learned on live school accounts and verified on a test account.

## Pages

| Purpose | URL |
|---|---|
| Wizard (8 steps: settings, duty types, staff, substitutes, review/generate, results, official form, notifications) | `/pro/pro_duty_roster/` |
| Saved plans (الجداول المحفوظة) | `/pro/pro_duty_roster/plans/` |
| Official form = what teachers see (the ACTIVE plan) | `/pro/pro_duty_roster/official-form/?start=YYYY-MM-DD&end=YYYY-MM-DD` |
| Daily attendance (تحضير المناوبين) | `/pro/pro_duty_roster/attendance/` |
| Monthly report | `/pro/pro_duty_roster/report/?year=&month=` |

Each saved plan card on the plans page has three forms: `/plans/<id>/activate/` (button `فتح وتعديل` on the active plan, `اعتماد وفتح` on others — opens it in the wizard AND makes it active), `/plans/<id>/open/` (`فتح بلا اعتماد` — open without activating) and `/plans/<id>/delete/`. Card text gives the period label and counts: `N تكليف · N مناوبة · N إشراف · N يوم عمل`. Fastest health check of any plan.

## Data model

The wizard page embeds the whole draft as JSON in `<script id="dr-initial-state">`:

```
settings     scheduleMode (term | monthly | weekly), termStartDate/termEndDate, monthlyStartDate/EndDate,
             selectedDays, maxDaily, defaultMin/MaxWeekly, distributionPattern (spread|cluster),
             substituteCount, substitutePattern (pool|dedicated), conflictSource, separateDailyCategories …
duties[]     {id, name, icon, category: duty|supervision, subtype (start_of_day, end_of_day, break …),
              active, start "06:30", end "07:00", required, locations[]}
staff[]      {id, name, categories[duty,supervision], minWeekly, maxWeekly, unavailableDays[], stage,
              preferredTypes[], excludedTypes[], isCustomized}
occurrences[] {key, dayId, dayName, date, hijri}         — the days the draft covers
schedule[]   {id "<date>-<dutyId>", key, dayId, date, dutyId, start, end, category, required, locations,
              assigned[staffIds], assignedLocations{staffId:[locations]}, locked, notes}
substitutes[] {key, date, dutyId|null, staffId}
holidays[], locations[], quality{total, coverage, fairness, categoryFairness, limitViolations …}
openPlan {id, title, isActive}     draftSync {status: synced | out_of_sync | saved_not_active | empty, planId …}
```

`state` inside the wizard's `app.js` is closure-scoped (not on `window`) — read the JSON from the DOM (or `fetch('/pro/pro_duty_roster/')` + `DOMParser`) and call the APIs yourself.

## APIs (POST JSON, header `X-CSRFToken` = `meta[name="csrf-token"]`)

| Endpoint | Body | Effect |
|---|---|---|
| `api/save-state/` | `{settings, duties, staff, locations, holidays, schedule}` | Saves the draft. Returns `{success, state, rejected[], warnings[], idMap, draftSync}`. New duty ids like `duty-<Date.now()>` come back mapped to real ids in `idMap.duties`, and the server creates one empty row per occurrence for the new duty. Substitutes are NOT part of the body and are left untouched. |
| `api/save-plan/` | `{title, openPlanId}` | Copies the draft into the plan `openPlanId` (= `state.openPlan.id`; the server rejects a mismatch as a conflict). With `openPlanId: null` on an unlinked draft it creates a NEW, unapproved plan. |
| `api/generate/` | none | Distributes from the saved draft. **It also saves AND ACTIVATES a plan** (response has `plan` with `isActive:true`) — never call it on a school account without explicit approval. |
| `api/new-schedule/` | none | Unlinks the draft from any plan and starts a fresh one (saved plans are untouched; duty types and staff stay). |
| `api/activate-open-plan/`, `api/discard-changes/`, `api/reset-schedule/` `{scope}`, `api/hide-staff/` | | as named |
| Attendance: `api/toggle-attendance/`, `api/mark-all-attended/`, `api/send-manual-notification/` | | attendance page only |

## Rules that bite

1. **One draft per school.** Opening any plan (`فتح وتعديل` / `فتح بلا اعتماد` / `اعتماد وفتح`) replaces whatever is in the wizard. Unsaved work — e.g. a duty type added after the last save — disappears. This is the usual cause of "جدول المناوبة اختفى": the plan is still on the plans page, and the wizard just shows another one.
2. **Duty types and staff settings are school-level**, shared by every draft and plan. Adding a type or changing a teacher's categories affects all plans.
3. **130-day window.** The wizard loads at most 130 occurrences, even when the term or plan is longer (a 165-day plan showed 2026-08-30 → 2027-03-31 only). Exports (`تصدير Excel/Word`) contain only those 130 days.
4. **Saving and generating touch only the window.** `save-plan` and `generate` replace the days inside the current window and keep every other day of the plan. Proven on a test account: a 220-day plan stayed at 220 days after editing the first 130 days + `save-plan`. A second generation with a later `termStartDate` *appends* days — that's how plans grow past 130.
5. **To edit days after day 130**, move the window: `save-state` with `termStartDate`/`termEndDate` covering those days (the plan's existing assignments load into the draft), edit, `save-plan`, then restore the original dates with another `save-state` + `save-plan` (the plan's period label follows the settings).
6. **Substitutes are generation-only.** No UI or API edits them (wizard step 6 and the attendance page show them read-only). They survive `save-state` / `save-plan`; a copied plan starts with none.
7. **Category check.** `save-state` rejects new or changed assignments whose teacher lacks the duty's category (`rejected[]` = `{slotId, dutyName, date, staff[]}`); unchanged assignments already in a plan are accepted as they are.
8. **Native confirm() freezes the extension.** Plan-card buttons use `form[data-confirm]` (delete of the active plan has a second `data-confirm-again`). Clicking them hangs the tab. Read the message from `form.dataset.confirm`, then submit without the dialog: `fetch(form.action,{method:'POST',body:new FormData(form)})` (the form carries the CSRF token), or `window.confirm=()=>true; form.requestSubmit()`. If a tab is already frozen, close it and continue in a new tab.
9. **The official form shows the ACTIVE plan**, whatever is open in the wizard. Use it to verify what teachers see. A draft change is invisible there until `save-plan`.

## Excel export format

`تصدير Excel` produces an HTML file named `.xls` (`table#officialTable`). Header cells are day names only (الأحد … الخميس, repeated per week — no dates). Weeks with holidays are shorter, so align columns with the plan's `occurrences` by sequence and check the weekday sequence matches. One `tbody tr` per duty type: first cell = type name + `div.dr-official-time` (category, time range); then one cell per day with `name<small class="dr-official-role"> (location)</small>`, `<br>` between people, and `<small class="dr-official-substitute">بديل: a، b</small>`.

## Recipes

### A. "The roster I made yesterday is gone"
1. Plans page: list every plan with its counts. Compare the wizard's `openPlan` with the active plan.
2. Parse the user's Excel (if any) and compare day by day with the official form of each candidate plan (hash `day|names` per day; see snippets).
3. Typical findings: the work is in the approved plan, but the wizard has another plan open; and/or one duty type is missing (it existed only in the unsaved draft).

### B. Restore or add a duty type and fill it from a file — no regeneration
1. Back up: note the plan's counts; save the days outside the window as text from the official form.
2. Open the plan (rule 8 technique) and confirm `draftSync.status === 'synced'`.
3. `save-state` with the extra duty → get its real id from `idMap.duties`.
4. Fill `assigned` + `assignedLocations` in that duty's rows (sorted by date, after checking the weekday sequence), then `save-state`. Require `rejected.length === 0` and the expected filled count.
5. `save-plan {title: unchanged, openPlanId: plan id}`. Expected: assignments = old + new, days unchanged.
6. Verify on the official form (all days, every duty type) and against the source file.

### C. Make a separate copy of a plan (leave the approved one untouched)
Snapshot the draft JSON → `api/new-schedule/` → `save-state` with the snapshot (fix staff categories if anything is rejected) → `save-plan {title:'… نسخة', openPlanId:null}`. The copy has ≤ 130 days and no substitutes. Activating it later would show teachers only those days.

### D. "Why is the distribution unbalanced?" (لماذا التوزيع غير متوازن)
Count per teacher × duty type over the whole plan (official form), and read staff settings. Usual causes:
- A duty type staffed by a fixed small group (e.g. morning gate by 4 people → about 40 shifts each vs about 25 for everyone else). It may be intentional — ask.
- Teachers with `maxWeekly 0` (exempt) or missing categories, or staff added after generation: they show 0 assignments.
- Different `required` per period (e.g. 4 supervisors in one part of the term, 3 later).
- `unavailableDays` not respected by manual entries — check before writing a rotation.
- The site's fairness % compares everyone in a category together, so a few exempt or specialised people drag it down even when the rest is even.

### E. Unsure what the server does? Test on a test account first
Ask the user for a test school account. Build the same situation (e.g. a plan longer than 130 days via two generations with moving `termStartDate`), run the exact sequence, compare the plans-page counts before and after, then restore the test account (re-activate its original plan, delete the test plan).

## Snippets

Read the draft (any duty page):
```js
const st=JSON.parse(new DOMParser().parseFromString(await fetch('/pro/pro_duty_roster/').then(r=>r.text()),'text/html').getElementById('dr-initial-state').text.trim());
({open:st.openPlan, sync:st.draftSync.status, occ:st.occurrences.length, first:st.occurrences[0]?.date, last:st.occurrences.at(-1)?.date,
  duties:st.duties.map(d=>d.id+':'+d.name), filled:Object.fromEntries(st.duties.map(d=>[d.id,st.schedule.filter(r=>r.dutyId===d.id&&r.assigned.length).length])), subs:st.substitutes.length})
```

Save draft / save plan:
```js
const H={'Content-Type':'application/json','X-CSRFToken':document.querySelector('meta[name="csrf-token"]').content};
const body=s=>JSON.stringify({settings:s.settings,duties:s.duties,staff:s.staff,locations:s.locations||[],holidays:s.holidays||[],schedule:s.schedule});
const sv=await fetch('/pro/pro_duty_roster/api/save-state/',{method:'POST',headers:H,body:body(st)}).then(r=>r.json());
// check sv.success, sv.rejected.length===0, counts in sv.state — THEN:
const sp=await fetch('/pro/pro_duty_roster/api/save-plan/',{method:'POST',headers:H,body:JSON.stringify({title:st.openPlan.title,openPlanId:st.openPlan.id})}).then(r=>r.json());
```

Plan counts from the plans page:
```js
const d=new DOMParser().parseFromString(await fetch('/pro/pro_duty_roster/plans/').then(r=>r.text()),'text/html');
d.body.innerText.replace(/\s+/g,' ').match(/الرجوع لمعالج المناوبة.*?جميع الحقوق/)[0]
```

Parse the official form (open it in the tab first — it needs layout, so `innerText`, not DOMParser):
```js
const days=[];let cur=null;
for(const tr of document.querySelectorAll('table tbody tr')){
  const m=tr.innerText.match(/(الأحد|الاثنين|الثلاثاء|الأربعاء|الخميس)\s*\n?\s*(\d+ [^\d\n]+ 14\d\d)/);
  if(m){cur={d:m[1],h:m[2],byType:{}};days.push(cur);} if(!cur) continue;
  for(const td of tr.querySelectorAll('td')){const t=td.innerText.trim().split('\n').map(x=>x.trim()).filter(Boolean);
    if(t.length>=2&&/^(مناوبة|اشراف|إشراف)/.test(t[1])) (cur.byType[t[1].split(' —')[0]]=cur.byType[t[1].split(' —')[0]]||[]).push(t[0]);}}
window.__D=days; days.length
```
Line 2 of each cell is the type label: the type name, or `مناوبة (نهاية الدوام (الانصراف)) — location` when the name is generic. Long outputs get truncated — return compact hashes (`djb2` over `day|names`) and compare them locally.
