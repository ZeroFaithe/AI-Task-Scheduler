# AI Academic Task Planner — Base Claude Skills

These are standalone skill files for use with **base Claude** (claude.ai) via the file upload feature. They are reformatted versions of the original Claude Code skills, translated into pure natural-language instructions that Claude can follow without running any code or accessing the file system.

---

## Files

| File | Role |
|---|---|
| `orchestrator-SKILL.md` | Coordinates the full pipeline: Planner → Scheduler → Reviewer |
| `planner-SKILL.md` | Validates tasks, assigns priorities, groups and sorts by urgency |
| `scheduler-SKILL.md` | Places tasks into concrete dated time slots, flags at-risk tasks |
| `reviewer-SKILL.md` | Checks the schedule, handles edits, generates final outputs on approval |

---

## How to Use

### Option A — Full pipeline (recommended)

1. Go to [claude.ai](https://claude.ai) and start a new conversation.
2. Upload **`orchestrator-SKILL.md`** using the file/attachment button.
3. Paste your task list (see format below) into the chat.
4. Say: **"Run the planner pipeline."**

Claude will step through Planner → Scheduler → Reviewer automatically, ask for your approval, and produce the final study plan.

### Option B — Individual skills

Upload only the skill you need if you want to run a single stage:

- **Planner only** — upload `planner-SKILL.md`, paste your task list
- **Scheduler only** — upload `scheduler-SKILL.md`, paste the Planner's output
- **Reviewer only** — upload `reviewer-SKILL.md`, paste the Scheduler's output

---

## Task List Format

Provide your tasks as a structured list (JSON, table, or plain text — Claude will parse any clear format). Each task needs:

| Field | Example | Notes |
|---|---|---|
| `subject_code` | `IPT10` | Short course code |
| `title` | `Lab 4` | Unique short name per subject |
| `task` | `Complete circuit diagram` | What needs to be done |
| `description` | `lab` | One of: `assignment`, `quiz`, `presentation`, `lab` |
| `est_hours` | `2.5` | Must be a multiple of 0.25 |
| `deadline` | `2026-11-05` | `YYYY-MM-DD` format |

Also include your **availability**:
- Hours available per day of the week (mon–sun)
- Earliest start time and latest end time (e.g. `08:00` – `22:00`)

Optionally include a `student_name` at the top.

**Example:**

```
Student: Maria Santos

Tasks:
1. IPT10 | Lab 4 | Complete circuit diagram | lab | 2.5h | due 2026-10-25
2. IPT10 | Midterm Quiz | Review chapters 1–5 | quiz | 3h | due 2026-10-28
3. ENG20 | Essay Draft | Write 1500-word draft | assignment | 4h | due 2026-11-02

Availability:
Mon 3h, Tue 2h, Wed 3h, Thu 2h, Fri 2h, Sat 5h, Sun 4h
Study window: 08:00 – 22:00
```

---

## Pipeline Overview

```
Your task list
      ↓
  [Planner]  — validates, assigns HIGH / MEDIUM / LOW priority, sorts by urgency
      ↓
 [Scheduler] — fills time slots day by day, flags at-risk tasks
      ↓
  [Reviewer] — checks for overlaps, hour totals, deadlines, subject correctness
      ↓
  Your approval
      ↓
  Final outputs: implementation.md + study calendar
```

### Priority thresholds (Planner)

| Priority | Rule |
|---|---|
| **High** | Due within 3 days; or quiz/presentation due within 7 days |
| **Medium** | Due within 10 days (beyond the high window) |
| **Low** | Due in more than 10 days |
| **Overdue** | Past due and not completed → always High |

To change these thresholds, tell Claude the new values at the start of your session.

### Revision logic (Orchestrator)

- If the Reviewer finds **issues** (conflicts, hour mismatches, deadline violations), the schedule is sent back to the Scheduler for revision — up to **3 rounds**.
- If the Reviewer finds only **warnings** (at-risk tasks), these are shown to you as informational. They do not block approval.
- After 3 unresolved revision rounds, Claude stops and shows you the remaining issues.

---

## Review-Phase Edits

After seeing the review result, you can ask Claude to change any task field:

> "Make Lab 4 three hours."
> "Change the Essay Draft deadline to November 10."
> "Mark Midterm Quiz as completed."

Claude validates the change, applies it, and reruns the full pipeline to show you what changed.
