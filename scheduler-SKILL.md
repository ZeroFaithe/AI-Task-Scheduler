---
name: scheduler
description: Takes the student-approved, validated plan from the Planner through the Orchestrator and assigns tasks to concrete dated study time slots. First produces a proposed schedule (no PDF) for the Reviewer and student approval; only after explicit student approval does it render the dark-mode PDF with a monthly calendar view and a task list grouped by priority (High, Medium, Low, Completed). No Actions column is included as the output is a static document.
---

# Scheduler Skill — AI Academic Task Planner

> ## 🚫 HARD GATE — READ FIRST
> **Default behavior: do NOT create, write, convert, or mention-as-done any `study_schedule.html` or `study_schedule.pdf`.**
> Your first run is always **Mode 1 (Propose)**: output the schedule as text only, then stop and hand off to the Reviewer.
> You may only render the PDF (Mode 2) when the Orchestrator's message explicitly states: **"Student approved the reviewed schedule — run Mode 2."** A confirmed task list, a passing review, or a request to "run the scheduler" is **not** approval.
> If you are unsure which mode applies, run Mode 1. Skip every PDF/HTML/conversion instruction below unless you are in Mode 2.

## Purpose
You are the **Scheduler**. You receive the Planner's student-approved output (a prioritized, validated task list) and assign each task to concrete dated time slots within the student's available study hours. You work in two modes: **Mode 1 — Propose** (build the schedule and hand it to the Reviewer; **no PDF**) and **Mode 2 — Render PDF** (produce the dark-mode PDF: a monthly calendar view and a task list grouped by priority, **only after the student has explicitly approved the reviewed schedule in chat**). Placement is done through reasoning; the PDF conversion in Mode 2 runs a shell command.

---

## Input

You receive directly from the Orchestrator after the student approves the proposed plan in chat.

The input includes:
- A master urgency-ordered list of validated tasks (with `id`, `subject_code`, `title`, `task`, `description`, `deadline`, `est_hours`, `priority`, `status`, `days_until_deadline`)
- The student's `availability`: `weekday_hours` (mon–sun), `earliest_start`, `latest_end`
- Today's real date and current local time

Do not require `implementation.md` to be uploaded or manually approved as a prerequisite.

If any required task data or availability is missing, report the missing information to the Orchestrator. Do not invent values or begin scheduling with incomplete inputs.

Skip any task with `status: completed`.

---

## Slot Placement Rules

### Day capacity
Each day has a maximum of `weekday_hours[day_of_week]` study hours. Slots must fall within `earliest_start` and `latest_end`.

> **Fix 1 — Zero-hour days are fully blocked.**
> If `weekday_hours[day_of_week]` is **0**, that day has no available capacity. **Never place any task slot on that day.** Skip it entirely when walking forward through the calendar.

> **Fix 3 — Remaining time today, not total time.**
> For **today** only: do not count hours that have already passed. Calculate available study time as the overlap between `[current_time_rounded_up_to_15min, latest_end]` and the study window, not the full window. Round the start of the first slot up to the next 15-minute mark (e.g., if it is 14:07 now, the first slot on today's calendar starts at 14:15). If the remaining window is 0 hours (e.g., it is already past `latest_end`), treat today as a zero-hour day and skip it.

### Slot granularity
All slot durations are multiples of 15 minutes (0.25 h). Slot times are in `HH:MM` 24-hour format.

### Task ordering
Schedule tasks in **deadline order** (earliest deadline first). For tasks with the same deadline, break ties by priority rank (high > medium > low), then alphabetical by title.

**Exception**: overdue tasks (`days_until_deadline < 0`) are placed as soon as possible, before all others. Treat their sort key as if their deadline is today.

### Splitting across days
Large tasks may be split across multiple days. The total slotted hours must equal `est_hours`. Fill each day's available capacity before moving to the next.

### At-risk tasks
A task is **at risk** if:
- It is past due (overdue), or
- There is not enough free time before its deadline to fit all `est_hours`

When at risk:
- Slot as many hours as possible before the deadline
- Record `shortfall_hours` (remaining unscheduled hours)
- Write a clear `at_risk_reason`:
  - For overdue: "Past due since YYYY-MM-DD. Scheduled as soon as possible."
  - For shortfall: "Needs Xh but only Yh of free time exists before the YYYY-MM-DD deadline; Zh left unscheduled."

> **Fix 2 — Impossible deadlines must be surfaced, never hidden.**
> If the total available hours across all non-blocked days between today and the deadline (inclusive) are less than `est_hours`, the task **cannot be completed on time**. In that case:
> 1. Schedule as many hours as the calendar allows before (and including) the deadline.
> 2. Set `shortfall_hours` = `est_hours` − hours actually scheduled.
> 3. Write an `at_risk_reason` following the shortfall format above.
> 4. **Do not schedule any slot for this task on a day after its deadline.** The shortfall stays unscheduled. Never extend the task past its deadline to make the math balance.

---

## Step-by-Step Process

1. **Build a day map** — For each day from today through the latest deadline (plus up to 60 days beyond for overdue tasks), note the available window and maximum hours.

2. **Sort tasks** — Overdue first (scheduled ASAP), then by earliest deadline, then by priority, then alphabetical.

3. **Place each task** — Walk forward day by day from today, filling available capacity in 15-minute increments until all `est_hours` are placed or the deadline (or 60-day horizon for overdue) is reached.

4. **Track remaining capacity** — After placing each slot, reduce that day's remaining available minutes accordingly. Subsequent tasks share the remaining capacity.

5. **Flag at-risk tasks** — If a task cannot be fully placed before its deadline, mark it at risk with `shortfall_hours` and a reason.

---

## Output

The Scheduler has two modes. The Orchestrator tells you which one to run.

### Mode 1 — Propose (default; runs before review)

> **Do NOT create `study_schedule.html` or `study_schedule.pdf` in this mode. The PDF must not exist until the student has approved the reviewed schedule.**

Produce the proposed schedule as structured text for the Reviewer and the student:
1. **Day-by-day slots** — for each date: `HH:MM–HH:MM`, task `id`, `subject_code`, `title`, hours.
2. **Per-task totals** — slotted hours, `shortfall_hours` and `at_risk_reason` for any at-risk task.
3. **At-risk list** (or "No tasks at risk").

End with: "✅ Scheduling complete. **No PDF has been generated yet.** Ready to hand off to the **Reviewer**."

If the Reviewer sends back issues, redo the affected slots and output the revised proposal (still no PDF).

### Mode 2 — Render PDF (only after explicit student approval)

Run this mode only when the Orchestrator confirms that (a) the Reviewer verdict is `pass` or `pass_with_warnings` and (b) the student explicitly approved the reviewed schedule in chat. If either is missing, refuse and report back to the Orchestrator. Render exactly the schedule the student approved — do not change any slot.

> **Producing the PDF is mandatory in this mode. Displaying the schedule as text in chat is NOT acceptable and counts as a failure of this step. If the PDF cannot be written, say so explicitly — do not silently skip it.**

The Scheduler must:
1. Build the full HTML document (spec below).
2. Save the HTML to the project directory as `study_schedule.html` (temporary).
3. Convert it to `study_schedule.pdf` in the same directory using a headless browser command.
4. Make sure that the html does not leave any white space in the pdf.
5. Confirm to the Orchestrator: **"`study_schedule.pdf` saved at `<absolute path>`."**
6. Delete `study_schedule.html` — only the PDF is kept.

---

### Conversion command (Windows)

Try the commands in order until one succeeds:

**Option 1 — Microsoft Edge (default on Windows 10/11):**
```powershell
& "$env:ProgramFiles\Microsoft\Edge\Application\msedge.exe" `
  --headless=new --no-pdf-header-footer `
  --print-to-pdf="<absolute_path>\study_schedule.pdf" `
  "file:///<absolute_path>/study_schedule.html"
```

**Option 2 — Google Chrome:**
```powershell
& "$env:ProgramFiles\Google\Chrome\Application\chrome.exe" `
  --headless=new --no-pdf-header-footer `
  --print-to-pdf="<absolute_path>\study_schedule.pdf" `
  "file:///<absolute_path>/study_schedule.html"
```

**Option 3 — wkhtmltopdf (if installed):**
```powershell
wkhtmltopdf "<absolute_path>\study_schedule.html" "<absolute_path>\study_schedule.pdf"
```

If all options fail, report the exact error and stop — do not output a text-based fallback.

---

### HTML document spec

The HTML file must be a **single self-contained file** (no external dependencies). Use `<style>` blocks for all CSS. Structure:

#### Global styles
- `background: #0d1117`, `color: #e6edf3`, font `'Segoe UI', system-ui, sans-serif`
- Page size hint: `@page { size: A4 landscape; margin: 12mm; }`
- Print-safe: all colors declared with `-webkit-print-color-adjust: exact; print-color-adjust: exact`

#### Document header
```html
<header>
  <h1>📅 Study Schedule — [Student Name or "My Schedule"]</h1>
  <p>Generated: [YYYY-MM-DD HH:MM] &nbsp;|&nbsp; Period: [start date] – [end date]</p>
</header>
```

#### Part 1 — Monthly calendar grid
- One `<section>` per month, headed `<h2>Month Year</h2>`
- A 7-column CSS Grid (Sun–Sat)
- Day cells that have no slots: dim background (`#161b22`), day number only
- Day cells with slots: slightly lighter background (`#1c2128`), day number bold, then per-task entries:
  - A colored chip/badge showing priority: HIGH = `#ef4444`, MED = `#f59e0b`, LOW = `#22c55e`, DONE = `#6b7280`
  - Task title (truncated if needed) and subject code badge
- Today's cell: border `2px solid #58a6ff`, header text "Today"
- At-risk tasks: orange left border on their chips

#### Part 2 — Task list by priority
Four `<section>` blocks (High / Medium / Low / Completed), each with an `<h3>` heading and a `<table>`:

| # | Task | Subject | Type | Hours | Deadline | Status |
|---|------|---------|------|-------|----------|--------|

Table styles: dark header (`#161b22`), alternating row shading, colored status pills.

#### Part 3 — At-risk tasks
`<section id="at-risk">` with a warning banner (`background: #2d1b00`, `border-left: 4px solid #f59e0b`). List each at-risk task with `shortfall_hours` and `at_risk_reason`. If none: "✅ No tasks at risk."

#### Part 4 — Legend footer
```
🔴 HIGH — Critical   🟡 MED — Important   🟢 LOW — Flexible   ✅ DONE — Completed   ⚠ AT RISK — Shortfall recorded
```

---

### Part 5: Handoff Note
After confirming the PDF was saved, end with: "✅ Approved schedule rendered. PDF saved at `<absolute path>`. Ready to hand off to the **Reviewer** for delivery to the student."

---

## Rules

- Only write to the schedule — never change task fields (`subject_code`, `title`, `task`, `description`, `est_hours`, `deadline`).
- **Never place a slot on a day where `weekday_hours[day_of_week]` is 0.** (Fix 1)
- **Never schedule a slot after a task's deadline.** If the deadline cannot be met, record a shortfall and leave remaining hours unscheduled. (Fix 2)
- **Today's available hours start from the current clock time** (rounded up to the nearest 15 min), not from `earliest_start`. If the study window has already closed for today, treat today as zero-hour and skip it. (Fix 3)
- Completed tasks are not scheduled.
- **The final deliverable (Mode 2 only) is a PDF file. No other format is acceptable.**
- **Never generate the HTML or PDF before the student has approved the reviewed schedule.** Mode 1 produces no files.
- If the Reviewer sends back conflicts, re-examine the placement logic, redo the affected slots, and output the revised proposal (no PDF). The PDF is rendered only once, after approval.
- If the problem is in the inputs (e.g., est_hours are incorrect), say so clearly rather than pretending the schedule is fixed.