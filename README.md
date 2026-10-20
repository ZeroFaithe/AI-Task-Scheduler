# AI Academic Task Planner — Base Claude Skills

These are standalone skill files for use with **base Claude** (claude.ai) via the file upload feature. Written as pure natural-language instructions that Claude follows through reasoning — no code execution or file system access required.

---

## Files

| File | Role |
|---|---|
| `orchestrator-SKILL.md` | Coordinates the full pipeline: Planner → (approval gate) → Scheduler → Reviewer |
| `planner-SKILL.md` | Clarifies missing fields, validates tasks, assigns priorities, produces `implementation.md` for student approval |
| `scheduler-SKILL.md` | Places tasks into time slots and produces a **dark-mode PDF-ready** calendar + priority task list |
| `reviewer-SKILL.md` | Runs 5 quality checks on the schedule, handles review edits, generates final outputs on approval |

---

## How to Use

### Option A — Full pipeline (recommended)

1. Go to [claude.ai](https://claude.ai) and start a new conversation.
2. Upload **all four skill files** using the file/attachment button.
3. Provide your input — even just a subject code and task name is enough to start.
4. Say: **"Run the planner pipeline."**

Claude will:
1. Ask you to clarify any missing fields (task type, estimated hours, deadline, availability)
2. Show you an `implementation.md` plan and **wait for your approval** before scheduling
3. Once approved, build the schedule and present a **dark-mode PDF-ready calendar** with a task list by priority
4. Run the Reviewer checks and ask for your final approval

### Option B — Individual skills

Upload only the skill you need if you want to run a single stage:

- **Planner only** — upload `planner-SKILL.md`, paste your task list (or just give a subject code + task)
- **Scheduler only** — upload `scheduler-SKILL.md`, paste the approved Planner output
- **Reviewer only** — upload `reviewer-SKILL.md`, paste the Scheduler's output

---

## Task Input Format

You can provide input at **any level of detail**. The Planner will ask for anything that's missing.

### Minimum (barebones) input
```
IPT10 — Lab 4
```
The Planner will prompt you for task type, estimated hours, deadline, and availability.

### Full structured input (recommended for speed)

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
Your input (barebones or full)
        ↓
   [Planner]  — clarifies missing fields, validates, assigns priorities
        ↓
   implementation.md  ← student reviews and approves this
        ↓
  ⛔ APPROVAL GATE — pipeline pauses here until you say "approve plan"
        ↓
  [Scheduler] — fills time slots, produces dark-mode PDF-ready calendar
        ↓
  [Reviewer]  — 5 quality checks; loops back to Scheduler on fail (max 3×)
        ↓
  Your final approval → Plan finalized
```

### Priority thresholds (Planner)

| Priority | Rule |
|---|---|
| **High** | Due within 3 days; or quiz/presentation due within 7 days |
| **Medium** | Due within 10 days (beyond the high window) |
| **Low** | Due in more than 10 days |
| **Overdue** | Past due and not completed → always High |

To change these thresholds, tell Claude the new values at the start of your session.

---

### Scheduler Output Format

The Scheduler produces a **dark-mode, PDF-ready document** with two sections:

1. **Monthly Calendar View** — a grid (Sun–Sat) showing each day's study tasks with priority color labels (`[HIGH]` · `[MED]` · `[LOW]`) and subject badges
2. **Task List by Priority** — a table grouped into High, Medium, Low, and Completed sections

> The task list does **not** include an Actions column — the output is a static, print-to-PDF document.

---

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
