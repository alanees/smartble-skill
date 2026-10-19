# Scheduling knowledge & lessons learned

## Vocabulary: assignment documents → site subject names
| Document (إسناد) | Site chip |
|---|---|
| لغتي | العربية |
| إسلامية | الإسلامية |
| رياضيات | الرياضيات |
| علوم | العلوم |
| انجليزي | الإنجليزية |
| اجتماعيات | الإجتماعية |
| أسرية / حياتية | الحياتية |
| مهارات رقمية | الرقمية |
| فنية | الفنية |
| بدنية | البدنية |
| نشاط | نشاط |

Grade words: أول/اول، ثاني، ثالث، رابع، خامس، سادس. Section numbers like `1+2+3` = sections 1, 2, 3. Site class names can contain spaces or typos (`حامس3` = خامس3) — normalise before matching.

Typical Saudi primary week: Sun–Tue 7 periods, Wed–Thu 6 periods. Default max per teacher 24.

## Reading assignment PDFs
- Extracted PDF text scrambles Arabic RTL — read the rendered page image.
- Verify each teacher: Σ(weekly periods of each class/subject) should equal `عدد الحصص`; the count of class-subject pairs ≈ `عدد الإسنادات`. Report mismatches; enter what the subject list says.
- Watch for the same class/subject listed under two teachers — the site will refuse the second; ask the user who keeps it.
- Teachers on leave/secondment (انتداب، إجازة مرضية) may be absent from the site; ask before creating them or assigning to placeholders (س١, س٢…).

## Why lessons go missing (الجدول ناقص)
- The generator never leaves a lesson "unplaced" visibly: if a class needs more periods than its week has, it fills every slot and quietly drops one lesson — a different subject in each class. Symptoms reported by staff: "I assigned it but it isn't in the table", "teacher X is short one period".
- Always check Σ(assignment periods per class) = slots per class before generating. Intermediate-school example: grades 1–2 = 34 periods (Arabic 4), grade 3 = 35 (extra التفكير الناقد 2, اجتماعيات 2); week Sun–Wed 7 + Thu 6 = 34 → every grade-3 class lost one lesson. Fix chosen by the school: default timing Thu 6 for grades 1–2, custom timing card 7×5 for grade 3.
- A zero-slack grid (lessons = slots, every class full) plus an over-full grade forced the solver to break "one subject per day": 49 class-day repeats (some subjects 3× in a day). After fixing capacity and raising `المادة مرة واحدة في اليوم` + `توزيع متوازن على أيام الأسبوع` to عالي (and `عدالة الحصص الأخيرة` + `توازن نصفَي اليوم` to منخفض): 0 repeats, 0 dropped, max 5/day, score 89 → 94.
- When staff say they "reduced" a subject (خفّضت الأنصبة), verify which subjects actually changed in the site — e.g. they said Arabic and Islamic went 5 → 4, but only Arabic had changed.

Typical Saudi intermediate (متوسط) week: 7 periods Sun–Wed, Thursday 6 or 7; نشاط often pinned at one period for all classes (e.g. Tue P4); math teachers may have period 7 closed (math as an early subject).

## Why quality hits a ceiling
- Subjects pinned to a period (e.g. نشاط for grades 1–3 at period 7 Sun–Tue) force those teachers into the last period → low "عدالة الحصص الأخيرة", and conflict with closed periods (a teacher with period 7 closed teaching a pinned نشاط). Fix the pin or the closure, not the priorities.
- Subjects taught every day (إسلامية 5/week, العربية 5–7/week) cannot move between days without duplicating — single-period swaps rarely exist for those teachers.
- Doubles (حصص زوجية, e.g. فنية 2 consecutive) move only as a pair and need two adjacent free periods.
- A teacher with 5 lessons on a 6-period day will inevitably have long consecutive chains.
- Raising "توزيع متوازن على أيام الأسبوع"/"الحد الأقصى للحصص اليومية" to عالي tends to shift peaks to other teachers rather than remove them. Disabling never-configured constraints (المواد المبكرة، إبعاد المواد، ترتيب المواد، التنقل بين المباني) changes nothing.
- Disabling soft constraints the school doesn't care about (e.g. عدالة الحصص الأخيرة، توازن نصفي اليوم، الحد الأقصى للحصص اليومية) gave the best result in practice (score 92 → 95, doubles 9/9, fewer long chains). The generator is non-deterministic — regenerate with a backup and compare.
- Closing a teacher's specific periods (تخصيص المعلم) is the most reliable way to force a day lighter; it persists for future generations, so tell the user.

## Decision etiquette
- Ask before: deleting teachers, wiping assignments, raising max periods beyond policy, regenerating (it reshuffles everyone), splitting doubles, accepting non-"consecutive" swap warnings.
- Report source-document conflicts and leave the choice to the user.
- When the user changes settings themselves, re-read and ask about anything that looks risky (e.g. a nursing-hour period reopened) instead of reverting it.

## Substitution (انتظار) knowledge
- Quota "auto" = max periods (24) − teaching periods; full-load teachers get 0.
- Typical exclusions to ask about: teachers the assignment PDF marks "لا يسند لها نشاط ولا مناوبة ولا انتظار" (e.g. gifted-programme teacher), reduced-load teachers (تخفيض نصاب), placeholders (س١…).
- Closed periods must apply to all teachers (nursing hours), not just seconded ones (site default).
- Some slots can't be filled: during a school-wide نشاط period every quota teacher is in class; late periods are blocked by "no substitution after the teacher's last lesson". Explain instead of overriding.
- Moving lessons later can collide with existing substitution duties — re-check and replace the substitute in that slot.

## Lessons from real schools (anonymised)
- **Timing-card names are only labels.** A custom card named «توقيت تلقائي 37 حصص» held the تحفيظ classes but its day counts were 7/7/7/6/6 = 33, so 4 lessons were silently dropped in every one of those classes. Always read the card's per-day counts, never trust its name.
- **The quality page does not detect dropped lessons.** It showed 91–92 and «0 حصص فارغة» while 12–24 lessons were missing. Always run the assignment-vs-table audit (js-snippets §4b).
- **A custom card is saved with the card's own button «حفظ التوقيت».** The sticky «حفظ جميع التعديلات» saves only the default timing, and it reports success while the card stays unchanged.
- **Changing تخصيص المواد** (`تطبيق التغييرات` → modal «نعم، تحديث») also re-aligns the workdays.
- **Stale locks survive.** Pinned نشاط for classes that no longer have نشاط (0 assigned), or locks on the wrong subject, block the solver. Unlock with a JS click on `.btn-lock-cell` (it persists). After generating, re-check every pinned lesson: one lock was lost during generation.
- **Edit-page select changes** persist only with a native `dispatchEvent(new Event('change',{bubbles:true}))`; jQuery `.trigger` alone does not save. Tick `#ignore-conflict-manual` first. Changing a cell this way needs the user's explicit OK, so prefer swaps.
- **After «تأكيد التبديل»** close the success modal («إغلاق») before the next swap, or the next clicks land behind it. A double-period half can't be a swap target.
- **The edit page hides the sidebar**, so guard the account name with `document.body.textContent`, not `innerText`. Fetched (DOMParser) edit-page HTML has empty option ids, so run clash checks by teacher id on the live DOM (`selectedOptions[0].id`).
- **Table history** `/pro/pro_dashboard/table-history/` shows the school's own edits; restoring creates tables named «… - نسخة تم إرجاعها …».
- **Schools clean up tables**, including your backups. Tell the school a backup exists before they tidy up, and give it a clearly protected name.
- **A school rule seen often:** the نشاط teacher of a class must also teach another subject in the same grade. Principals accept exceptions, so report violations and don't force them. نشاط is usually locked at the same period for all classes, so one teacher can hold at most one نشاط per locked period.
- **Typical staff messages are relayed and imprecise** (wrong subject for a teacher, "reduced" subjects that weren't changed). Verify each claim against the data.
