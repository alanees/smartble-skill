# Smartble Pro — site map, selectors, click quirks

Base: `https://www.smartble.net`

## Pages

| Purpose | URL |
|---|---|
| Teaching assignments list (العلاقات التدريسية) | `/pro/pro_dashboard/assignments/?per_page=100&page=1` |
| One teacher's assignments | `/pro/pro_dashboard/teacher-assignments/<teacherId>/` |
| Teachers (add / delete) | `/pro/pro_dashboard/teachers/?per_page=100` |
| Teacher customize (max periods, closed periods) | `/pro/pro_dashboard/teacher-customize/<teacherId>/` |
| Timetable preview + auto-generate | `/pro/pro_smart_table/table-preview/` |
| Manual edit / swaps | `/pro/pro_dashboard/table/` |
| Teacher-to-teacher swap tool | `/pro/pro_smart_table/teacher-swap/` |
| Analysis | `/pro/pro_dashboard/table-analysis/` |
| Quality report (جودة الجدول) | `/pro/pro_dashboard/table-quality/` |
| Constraint priorities (تخصيص القيود) | `/pro/pro_dashboard/solver-priorities/` |
| Advanced / all tables | `/pro/pro_dashboard/advanced/` |
| All assignments in one page (fast read/diff) | `/pro/pro_dashboard/print-teacher-assignments/` |
| Subject settings per class (تخصيص المواد) | `/pro/pro_dashboard/courses/` → `/pro/pro_dashboard/course-customize/<courseId>/` |
| Substitution hub (حصص الانتظار) | `/pro/pro_waiting_classes/` |
| Substitution wizard | `/pro/pro_waiting_classes/wizard/` (`?step=1..4`, `?completed=1` on success) |
| Substitution quotas per teacher | `/pro/pro_waiting_classes/teacher-waiting-classes/` |
| Fixed substitution schedule (edit) | `/pro/pro_waiting_classes/edit-wc-table/` |
| Duty & supervision roster wizard (المناوبة والإشراف) | `/pro/pro_duty_roster/` — details in duty-roster.md |
| Saved duty plans / official form / attendance | `/pro/pro_duty_roster/plans/`, `/official-form/?start=&end=`, `/attendance/` |

Login page after `تسجيل الخروج` is `/home/` — the user must type credentials themselves.

If `navigate` fails with "Could not verify this site's safety category", retry once, or navigate from inside the page: `javascript_tool` → `location.href='/pro/...'`.

## Workdays (الأيام الدراسية) — `/pro/pro_dashboard/workdays/`
- Settings are per table (header shows the table name). Default timing card `توقيت الحصص` has two modes: `button.wd-mode-btn` `نفس التوقيت لجميع الأيام` (one period count for every day) and `تخصيص يومي` (day tabs الأحد…الخميس, each with its own count and +/− buttons). Per-day counts may already be stored while "same for all days" is active — switching to `تخصيص يومي` reveals them (Thursday showed 6).
- `توقيت الحصص المخصص` cards override the default for selected classes (chips under `الفصول الدراسية المعينة`, e.g. «توقيت تلقائي 35 حصص» on the five ثالث classes). Expand with the chevron on the card's left.
- Save: sticky bar `تغييرات غير محفوظة` → `حفظ جميع التعديلات` (real click) → modal `تأكيد حفظ التعديلات` (warns existing table data may be affected) → `تأكيد`. Verify on table-preview: removed periods appear as `td.is-placeholder`.

## Subject settings (تخصيص المواد) — `/pro/pro_dashboard/courses/`
Class selector + one row per subject: teacher, `input[name=odd_classes]` (single periods), `input[name=even_classes]` (number of doubles), `early_class` checkbox, special room select. Weekly periods = odd + 2×even (e.g. رياضيات 4+1 = 6). Subject names differ between tables (one table had الرياضيات/العربية/الإنجليزية, its successor رياضيات/عربي/انجليزي) — never hard-code them.

## Assignments list
- Teacher IDs: `[data-teacher-id][data-teacher-name]` (on the clear buttons).
- Current table selector: `#table-select` (form POSTs to the same page; submit with the button `اختر الجدول`). Option text includes `(الحالي)` for current; other tables (backups, older terms) are listed too.
- Per row: `.asg-clear-toggle` (حذف الإسنادات) → `.clear-without-btn` (حذف بدون إعادة توزيع) opens a modal; confirm button text `نعم، مسح دون إعادة تعيين`. JS `.click()` on these works.
- `.reassign-option` = حذف مع إعادة توزيع (auto redistribution — avoid).

## Teacher assignments page
- Rows: `td.ta-classroom-col` (class name) + `button.ta-course-btn[data-course-id]`; subject = first text node, badge `(n)` = weekly periods.
- Chip states: `is-free` (unassigned), `is-self` (this teacher), `is-other` (another teacher). Selected adds `is-selected`.
- Submit: `#btn-assign` (تعيين المحدد). JS click works but the page then navigates, so trigger it with `setTimeout(...,100)` and return results first; wait ~3 s before the next action. Success banner: `تم الإسناد بنجاح !`.
- Conflict mode select `#ignore-conflicts` (0 = check & skip conflicts — keep default).
- Stats text: `أقصى عدد للحصص`, `الحصص المسندة`, `متبقي`.
- Removing one: click the `is-self` chip → toolbar `حذف المحدد` → modal `نعم، إزالة الإسناد` (use real clicks).

## Teachers page
- Add: `#id_name` (required), `#id_phone_number`, `#id_group`, submit button text `أضف`. Banner `تم إضافة معلم جديد: ...`.
- Delete: `.btn-delete-teacher-dropdown[data-teacher-id]` → confirmation modal `تأكيد الحذف` → button `احذف` (JS click opens the modal; press the modal button with a real click; verify by reloading the list).
- Customize button: `.btn-customize[data-teacher-id]`.

## Teacher customize page
- `input[name=max_classes]` + button `احفظ` (top form).
- Availability grid: table with rows الأحد..الخميس; checkbox checked = available, unchecked = closed. Wed/Thu may have only 6 periods.
- Save: button `احفظ أوقات العمل المتاحة` — **needs a real mouse click at its coordinates**; ref-based and JS clicks did not persist. Always reload and re-read the grid.

## Table preview
- DOM holds all five days: class rows are `tr` whose first cell matches a class name; there are (#classes × 5) rows in day order, so `day = days[Math.floor(rowIndex / #classes)]`.
- Cell text: line 1 subject, line 2 teacher short name (the same short name can belong to two different teachers — don't merge them blindly). Empty cells show `—`; some rows (custom-timing classes) also carry a time line like `10:15 – 11:00` — filter lines matching `^\d+:\d+`. Closed/non-existent periods are `td.class-cell.is-placeholder` — skip them, they are not "empty".
- An ungenerated table shows only `—` cells plus locked lessons (e.g. نشاط pinned at the same period for all classes).
- Generate: button `بدء التوليد الآلي` (dropdown; the first click sometimes only focuses — click again), item `#start-auto-generate-btn` (ابدأ التوليد التلقائي الآن), then modal `تأكيد`. Progress banner at the top `جاري إنشاء الجدول تلقائياً` with %. `#analysis-btn` opens table-analysis in a new tab — navigate instead.
- `إدارة الجداول` menu: `إنشاء جدول جديد` (modal `#createTableModal`, select `table-copy-id` to copy an existing table), `اختيار جدول آخر`, `تعيين الجدول النشط`, `تغيير اسم الجدول`.

## Manual edit page (/pro/pro_dashboard/table/)
- Cells: `td[data-cell-day][data-cell-class][data-cell-classroom]` (cellClass = period number). Each has `select[name=course]`, lock buttons, and `.swap-button` (⇄).
- Only one day is visible; day tabs at the top (`الأحد … الخميس`), sticky when scrolled. A tab click sometimes needs repeating — verify the visible day via `td.offsetParent!==null`.
- Swaps need **real clicks**: click source ⇄ → switch day → click the orange ⇄ on the target cell → preview modal `تبديل الحصص` with `تحذيرات` → `تأكيد التبديل`, or ✕ to cancel. Success: `تم التبديل بنجاح!`.
- JS `.click()` on swap buttons does nothing. The `find` tool fails here (page too large) — use JS to scroll the cell into view and compute coordinates.

- Each `select[name=course]` option carries the teacher id in its `id` attribute and text `nickname - subject`; cells with `data-is-off-cell="true"` are not real periods. Locked cells show `.btn-lock-cell` without `hidden` (e.g. نشاط fixed at the same time for all classes).

## Print assignments page
`/pro/pro_dashboard/print-teacher-assignments/`: one table row per teacher — cell 1 full name, cell 2 items like `[الرياضيات - رابع 3]`, cell 3 periods, cell 4 count. Fastest way to read every assignment.

## Substitution (انتظار) pages
- Wizard step 2: radios `quotaOptions` (`auto`/`manual`); `حفظ واستمرار` on auto opens modal `تأكيد الحساب التلقائي` → `نعم، تابع`.
- Wizard step 3: one `input[type=number]` per teacher (name `wc-<id>`), label shows current/max periods.
- Wizard step 4 (resets to defaults each visit): select with options `عدم التطبيق / تطبيق للمعلمين المنتدبين فقط / تطبيق لجميع المعلمين`, `#waiting-teachers-per-class`, `#waiting-teachers-last-class`, `#repeatInDay`, `#noWcAfterLastClass`, per-slot number inputs (last period of each day defaults to none), button `بدء التوزيع` (real click).
- teacher-waiting-classes: `#update-list` builds the list (default 3 each), `#distribute-system` (0 متغير, 1 ثابت حسب الفصل والحصة, 2 ثابت حسب الحصة, 3 جداول متعددة). `#distribute-wc-count` (توزيع تلقائى) triggers a native confirm that freezes the tab — use the wizard instead.
- edit-wc-table: day rows × period cells; names in `.teacher-nickname`; button `تعديل قائمة الانتظار` opens `#wcAssignModal` with `select[name=wct-select]` (only free teachers), `select[name=wct-priority]`, `إضافة معلم انتظار` (real click), trash icon → modal `حذف معلم منتظر` → `تأكيد`. Closing the modal reloads the page.

## Coordinates
Screenshot frame width (e.g. 1475 or 1568) differs from `innerWidth` (e.g. 1627 or 1833) and changes between sessions — read both each time. Convert: `screenX = cssX * frameWidth / innerWidth` (same factor for Y). Scroll the element into view first.

## Freezes and dialogs
If `javascript_tool` times out ("renderer may be frozen"), a native alert/confirm is probably open on that tab. Pressing Enter via the extension doesn't reach it; ask the user to answer the dialog in Chrome, then continue. Avoid buttons known to use native confirm().

## Quality page
Score (`درجة الجودة`), `الأضعف مقارنة بما طلبته`, then metric cards with % and details. Click every element whose text is `N عنصر بحاجة إلى مراجعة` to expand the named items.

## Solver priorities page
Each constraint card: toggle checkbox (on/off), level buttons `منخفض / عادي / عالي` (selected = coloured background), up/down order arrows. A sticky bar `تغييرات غير محفوظة` appears with `احفظ` top-left; success `تم حفظ الأولويات`. Disabled cards move to the bottom; raised ones move to the top. JS `.click()` on the level buttons works (snippet §9b); the `احفظ` button needs a real click; reload and re-read with §9 to verify.
