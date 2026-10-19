---
name: smartble-skill
description: Operate the Smartble Pro school-timetable site (الجدول الذكي, smartble.net) through Claude in Chrome — entering or diffing teaching assignments (العلاقات التدريسية / إسنادات) against a PDF, adding/deleting teachers, nicknames (اسم الاختصار), closing teacher periods (تخصيص المعلم, ساعة رضاعة), raising max periods, copying/creating/activating tables (جدول جديد، اعتماد الجدول، تعيين الجدول النشط), auto-generating the timetable (التوليد الآلي), diagnosing an incomplete or messy timetable (الجدول ناقص، حصص ناقصة للمعلمات، تكرار المادة/الفصل في اليوم، ٦ حصص يوم و٢ يوم), editing school days and periods per day (الأيام الدراسية، عدد الحصص، حصة سابعة الخميس), reviewing جودة الجدول, tuning تخصيص القيود, moving or swapping a teacher's periods (انقل الحصة، تبديل الحصص، موازنة الأيام), building the substitution schedule (جدول الانتظار / حصص الانتظار / معالج الانتظار), and the duty & supervision roster (المناوبة والإشراف، جدول المناوبة، نموذج 8، المناوبة الصباحية، إشراف الفسحة، المناوبون البدلاء، تحضير المناوبين) — restoring a "lost" roster from its Excel export, adding a duty type without regenerating, editing days beyond the wizard's 130-day window, copying/approving roster plans, and explaining an unbalanced duty distribution. Use this skill whenever the user mentions smartble, الجدول الذكي, إسنادات المعلمات, العلاقات التدريسية, توليد الجدول, جودة الجدول, جدول الانتظار, المناوبة, الإشراف, or asks to fix/balance/move anything in a school timetable or duty roster on that site — even if they don't name the site explicitly.
---

# Smartble Pro (الجدول الذكي) operator

Smartble Pro is a Saudi school-timetable web app at `https://www.smartble.net/pro/...`. Users (usually school principals/schedulers writing in Arabic) ask you to load teacher assignments, generate the timetable, and improve its quality. You drive the site through the **Claude in Chrome** MCP tools. Reply to the user in Arabic when they write in Arabic.

Read `references/site-map.md` for URLs, DOM selectors and which clicks need real mouse events. Read `references/js-snippets.md` before automating assignments, audits or swap searches — it has tested JavaScript you can paste into `javascript_tool`. Read `references/scheduling-knowledge.md` before trying to improve quality or balance a teacher. Read `references/duty-roster.md` for anything about the duty & supervision roster (section 7). If `references/known-accounts.md` exists (a private, local-only file), read it for notes on schools you have worked with before.

## 1. Verify the account and table before ANY write

This matters more than anything else: the user may have several Chrome profiles logged into different schools, and the site keeps a "current table" (الجدول الحالي) that silently changes.

1. `list_connected_browsers`; several "Browser N" entries may exist. Open the site in the selected browser and read the profile name in the sidebar (the school or principal's name). If it shows another school, close that tab, `select_browser` another deviceId, and re-check. If unsure which account is right, take a read-only screenshot of the user's own Chrome window (computer-use, browser = read tier) to see what they have open.
2. Read the current table: on `/pro/pro_dashboard/assignments/` the `#table-select` shows `(الحالي)`; the header also shows the table name and whether it is active (`هذا الجدول غير نشط`). Note that "current" (where edits go) ≠ "active" (published).
3. Re-check the account name at the start of every new tab/turn — guard writes with `if(!/ACCOUNT NAME/.test(document.body.innerText)) ...`.
4. Switching accounts: you may click `تسجيل الخروج` when asked, but never type usernames/passwords — leave the login page open and ask the user to sign in, then re-verify the profile name and current table.
5. Tables come and go between sessions (the user creates new ones). Always read `#table-select` fresh; don't assume the table from a previous session is still current or active.
6. **Writes need the user's explicit go-ahead.** The harness's auto-mode classifier blocks saves on a school account (workdays, priorities, generation…) until the user approves in chat. Do all read-only diagnosis first, then list the exact writes in a short table (what, where, expected effect) and ask once; after "موافق" do them in order and verify each.
7. The user often relays messages from school staff (a وكيلة or an admin assistant who isn't a timetable specialist). Their wording is vague and their facts can be off (wrong subject for a teacher, a teacher who "lost a period" but didn't). Translate each complaint into a checkable claim, verify it against the data, and report what the data says — including where the staff member was mistaken.

## 2. Entering teaching assignments from a document

1. Read the source (PDF extracted text is often garbled RTL — trust the rendered page image over the extracted text). For each teacher, list subject × grade × section numbers, and sanity-check against the stated period total (sum of weekly periods per class/subject) and assignment count. Note discrepancies in the source rather than silently "fixing" them.
2. Map document vocabulary to site subject names (see scheduling-knowledge.md: لغتي→العربية, أسرية→الحياتية, مهارات رقمية→الرقمية, اجتماعيات→الإجتماعية, انجليزي→الإنجليزية…). Normalise class names (spaces, typos like `حامس3`).
3. Get teacher IDs from the assignments list (`?per_page=100`, `[data-teacher-id][data-teacher-name]`). Teachers missing from the site (e.g. on leave) — tell the user instead of guessing.
4. For each teacher open `/pro/pro_dashboard/teacher-assignments/<id>/`, select chips with the assignment snippet, and submit only if every wanted chip was `is-free` and found. A chip owned by another teacher means the source double-books it — stop and report it; don't override.
5. Batch several teachers per `browser_batch` (navigate → js → wait 3s). Afterwards reload the list and compare each teacher's period total with the source.
6. Gotcha: if a teacher would exceed `أقصى عدد للحصص` (default 24) the assign silently does nothing. Check `الحصص المسندة` after submit; raise `max_classes` on the teacher-customize page only if the user agrees, and put it back when no longer needed.
7. Final completeness check: on any teacher-assignments page count `button.ta-course-btn.is-free` — 0 means every class/subject has a teacher.
8. **Updating from a revised PDF:** don't re-enter everything. Read all current assignments at once from `/pro/pro_dashboard/print-teacher-assignments/` (diff snippet in js-snippets.md), encode the PDF as a spec, and diff: per-teacher adds/removes, class-subjects claimed by two teachers in the PDF, and class-subjects no one claims. Apply only the real differences (remove from the old teacher, assign to the new one) and report PDF typos instead of applying them blindly.
9. School-specific assignment rules (e.g. "the نشاط teacher of a class must teach another subject in the same grade") are checks to report, not to enforce — the principal may accept exceptions. Check, list violations with candidate fixes, and let the user decide.

## 3. Teacher maintenance

- Add teacher: teachers page form (`#id_name` + button `أضف`). Delete: row dropdown `احذف` → modal `احذف` (needs a real click). Placeholder teachers like `س١`, `س٢` are common for vacant loads.
- Move a teacher's whole load: assignments list → `حذف الإسنادات` → **`حذف بدون إعادة توزيع`** (confirm `نعم، مسح دون إعادة تعيين`), then assign to the new teacher. Avoid `حذف مع إعادة توزيع` — it auto-redistributes to teachers the site picks.
- Remove one assignment: on the teacher page click the `is-self` chip → `حذف المحدد` → confirm `نعم، إزالة الإسناد`.
- Close periods (e.g. ساعة رضاعة = last period each day; Wed/Thu may have only 6 periods): teacher-customize grid, checked = available. Save with a real click on `احفظ أوقات العمل المتاحة`, then reload to verify. Closures and the period grid can differ per table (a copied table had the closure shifted by one period) — re-read them in the current table.
- Nicknames (`اسم الاختصار`, `input[name=nickname]` on teacher-customize, saved with the top `احفظ`) are what the timetable, edit page and substitution schedule display. Watch for duplicates or wrong nicknames (two different teachers both shown as the same short name; one teacher's nickname set to another teacher's name) — they make printed schedules ambiguous and break name-based audits. Use the teacher id (option `id` on the edit page) to disambiguate, and offer to fix bad nicknames.

## 4. Generating and reviewing

- Before regenerating, offer a backup: `إدارة الجداول → إنشاء جدول جديد` with `نسخ من جدول آخر`. Creating it makes the copy the current table — if the copy is only a backup, immediately switch back via the `#table-select` form on the assignments page and verify; if the user wants to work in the new copy (e.g. "انشئ جدول جديد منسوخ من الحالي"), keep it current. Verify the copy by comparing filled/empty cell counts with the source.
- Activating (`اعتمد الجدول`): `إدارة الجداول → تعيين الجدول النشط` → select the table in `#setActiveTableModal` (`select[name=table_id]`) → `تنشيط الجدول`. Verify the option now reads `(الجدول النشط) (الحالي)`. Only do this when asked.
- Generate: table-preview → `بدء التوليد الآلي` (may need two clicks to open) → `ابدأ التوليد التلقائي الآن` → `تأكيد`. Takes ~3–8 min (15 classes × 35 periods ran ~13 %/min); poll the top progress banner with `browser_batch` waits of ≤90 s per batch. Don't run a long `await` loop in `javascript_tool` — the page reloads when generation ends and the call fails with "Inspected target navigated"; that reload (banner gone) *is* the completion signal.
- Verify with the full audit (js-snippets §4b): every assigned lesson placed (assignment count = table count per class×subject), 0 empty non-closed cells, 0 same-subject repeats in a class-day, per-teacher day loads, and no lessons in closed periods.
- Quality: `/pro/pro_dashboard/table-quality/` (from `تحليل ومراجعة → جودة الجدول`). Expand items by clicking each `عنصر بحاجة إلى مراجعة`. Report score, weakest metrics, and named teachers.
- The user may change constraints/closures themselves between turns — re-read settings rather than assuming your last state, and ask if a change looks unintended (e.g. a nursing-hour period reopened).

## 4b. Diagnosing "الجدول ناقص" / repeats / uneven days

Typical complaint: "I entered all assignments but the table comes out incomplete, some teachers are short a period, the same class repeats in a day, a teacher has 6 one day and 2 another." Diagnose read-only before touching anything:

1. **Capacity per class** (js-snippets §4c): sum each class's assignment periods (teacher-assignments page badges) and compare with its available slots in the table (periods per day from the workdays settings, minus `is-placeholder` cells in table-preview). The generator **silently drops** lessons when a class needs more periods than it has — there are no empty cells, a different subject just goes missing in each class. A week with Thu 6 periods = 34 slots; a grade with an extra subject (e.g. التفكير الناقد) can need 35.
2. **Dropped lessons** (§4b): per class×subject, assignment count vs table count. Each missing lesson maps to a teacher — that is who "lost a period". Name them; check the staff's list against it.
3. **Repeats**: count class-day-subject occurrences > 1, excluding the intended double (حصة زوجية from تخصيص المواد: `even_classes`). Different subjects by the same teacher in one class-day are not repeats.
4. **Day loads** per teacher; small-load teachers (3–8 periods, or with closed days) are naturally uneven — don't flag them as problems.
5. Read the settings that drive it: workdays (periods per day, custom timing cards per class group), تخصيص المواد (`odd_classes`/`even_classes`/`early_class` per subject), teacher closures (§7b), solver priorities (§9).

Fixes, in order of impact (each needs the user's decision):
- Make each class's assignment total = its slots: reduce a subject's weekly periods (the school decides which), or change periods per day for that grade. Grades can have different weeks: keep the default timing for most classes and use a **custom timing card** (`توقيت الحصص المخصص`) for the grade that needs a longer week, or switch the default to `تخصيص يومي` and set Thursday to 6.
- With 0 slack (lessons = slots) the solver sacrifices "one subject per day": set `المادة مرة واحدة في اليوم` and `توزيع متوازن على أيام الأسبوع` to عالي, and `عدالة الحصص الأخيرة` + `توازن نصفَي اليوم` to منخفض. Result seen: 49 repeats → 0, score 89 → 94, max 5/day, 0 dropped lessons.
- A new, empty table can be generated without a backup (nothing to lose); the old table stays in `#table-select` as the fallback.

## 5. Balancing a teacher with manual swaps

1. Audit the teacher's week; identify the heavy day and light day.
2. Run the swap-search snippet: candidate = same class, teacher's lesson on heavy day ⇄ another lesson on a light day, with the other teacher free, no closed periods, no breaking a double (حصة زوجية), no subject exceeding its per-day cap, and the other teacher's day staying within 4–5. Try two-step chains if single swaps fail.
3. Execute in `/pro/pro_dashboard/table/` with real clicks: select day tab (may need two clicks), click the source cell's ⇄, switch day, click the orange target ⇄. A preview modal lists warnings.
4. Accept only warnings the user allowed (typically `حصص متتالية`). Close the modal (✕) for anything else — e.g. "teacher will have 6 periods", splitting a double, subject twice a day — and ask the user.
5. Re-check quality after swapping. If a swap made things worse, revert it with the same swap. Never leave a half-done chain.
6. If no clean swap exists, say so plainly, show the best options with their cost in a table, and let the user choose.
7. Explicit moves ("انقل الأولى يوم الخميس إلى الثالثة" for a teacher) are same-day swaps inside the teacher's class: check the teacher is free at the target period and the other teacher is free at the source period, then swap and confirm (consecutive-period warnings are fine for an explicit request).
8. Structural limits: if a short day (e.g. Thursday 5 periods) or an empty last period leaves fewer slots on some days, "3 periods on Thursday" flags are unavoidable — explain it rather than chasing them. Re-enabling `الحد الأقصى للحصص اليومية` helps against 6-period days.
9. **After any move/swap, check the substitution schedule** (section 6): a teacher who now teaches in a period where she was a substitute must be replaced there.

## 6. Substitution schedule (جدول الانتظار)

Prefer the wizard `/pro/pro_waiting_classes/wizard/` (4 steps); avoid the `توزيع تلقائى` button on `teacher-waiting-classes` — it opens a native confirm() that freezes the tab for the extension.
1. Ask the user first: quota rule (auto = max periods − teaching periods, or fixed), who is excluded (e.g. talent/gifted teacher per the assignment PDF notes, reduced-load teachers, placeholders س١…), whether closed periods apply to all teachers (they should, for nursing hours), and substitutes per period.
2. Step 1: active table + `ثابت حسب الحصة` → `حفظ واستمرار`. Step 2: `توزيع آلي` → confirm `نعم، تابع`. Step 3: set excluded teachers' inputs to 0 (native value setter + input/change events); total should cover (periods needing substitutes × per-period count). Step 4 settings reset every visit: set the closed-periods select to `تطبيق لجميع المعلمين`, `#waiting-teachers-per-class`, `#repeatInDay`, `#noWcAfterLastClass`; then real-click `بدء التوزيع` and confirm the page shows `?completed=1`.
3. Audit `/pro/pro_waiting_classes/edit-wc-table/` (`.teacher-nickname` per cell): periods with fewer substitutes than required, per-teacher counts. Unfilled slots are usually real (everyone with quota is teaching then, e.g. the نشاط period) — compute who is free and explain.
4. Manual fixes: `تعديل قائمة الانتظار` (JS click right after a fresh page load) opens `#wcAssignModal`; `select[name=wct-select]` lists only teachers free at that time; pick, then real-click `إضافة معلم انتظار`. Remove with the trash icon → `تأكيد`. Closing the modal reloads the page — handle one slot per load.

## 7. Duty & supervision roster (المناوبة والإشراف)

Read `references/duty-roster.md` first: pages, the draft JSON, the APIs and the rules below in detail.
1. **One draft per school.** The wizard holds a single draft; opening any saved plan replaces it, and unsaved work vanishes. "My roster disappeared" usually means the plan is still on `/pro/pro_duty_roster/plans/` and the wizard shows another plan, or a duty type existed only in the unsaved draft. Compare every plan's counts and the user's Excel export (HTML `.xls`, day-name columns only) day by day before changing anything.
2. **Never press generate on a school account without approval** — `api/generate/` saves AND activates a plan, and replaces the assignments and substitutes in its window. To add or restore a duty type, use `save-state` + `save-plan` (recipe B) and keep everything else as is.
3. **130-day window:** the wizard and its exports show at most 130 days. `save-plan` and `generate` only replace days inside the window, and the rest of the plan stays (tested). Edit later days by moving `termStartDate`/`termEndDate`, then restore the dates.
4. Plan-card buttons (`فتح وتعديل`, `اعتماد وفتح`, `حذف`) open a native confirm() that freezes the extension — read `form.dataset.confirm` and POST the form with `fetch` + `FormData` instead.
5. Substitutes can't be edited after generation (no UI or API); they survive saves. A copied plan has none.
6. Verify on the official form (`/pro/pro_duty_roster/official-form/?start=&end=`), which shows the ACTIVE plan — what teachers actually see: days, every duty type, counts on the plans page before/after.
7. When the server's behaviour is uncertain and the change is hard to undo, ask for a test account, reproduce the situation there, and only then touch the school's approved plan.
8. Balance questions: count per teacher × duty type and read staff settings (`categories`, `min/maxWeekly`, `unavailableDays`) — fixed small groups, exempt staff and late-added staff explain most of it (recipe D).

## Reporting style

Lead with the outcome, then a short table or bullets: what changed, what was verified, what is left and needs the user's decision. Be honest when something did not persist or could not be fixed. Close every tab you opened when done.
