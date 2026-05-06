# Notion EOD reports — how to use this skill

This skill tells the agent how to build **end-of-day (EOD) Hour Tracking reports** from your **Notion Master Tracker** plus **Time Doctor** exports. The agent uses the **Notion MCP** to read cards and comments; it must not invent Notion data.

## Quick start (what you say in chat)

Use a **short message** in a **fresh thread** when possible (fewer tokens, cleaner runs). Attach your two CSV files and include:

1. A **trigger** so the agent loads this workflow (e.g. “follow notion EOD reports” or “generate my reports”).
2. The **report date** — the calendar day the EOD is for.
3. Optionally a **lead name** — limits the report to that lead’s client roster (see below).
4. The two files: **tasks** export and **summary** export.

**Example (lead-filtered EOD for one day):**

```text
Can you please follow notion EOD reports and generate EOD reports of 05-05-2026 for Mayur.

@/path/to/summary-mayur.csv
@/path/to/tasks-mayur.csv
```

Replace paths with your real files (Cursor `@` file mentions work well).

### What each part means

| Piece | Role |
|--------|------|
| **Follow notion EOD reports** / **generate my reports** | Tells the agent to use this skill’s batch workflow. |
| **Date** (e.g. `05-05-2026`, `2026-05-05`, `5 May 2026`) | **Report day**: which day’s work the batch reflects (filters `tasks.csv` by **Date** when present) and, when Notes are included, which **`UPDATE (DD-MM-YYYY)`** pattern to match on the **master** Notion card’s comments. |
| **Lead name** (e.g. **Mayur**) | **Lead-filtered batch**: only **Project** rows whose client appears on that lead’s roster in the skill; other Time Doctor projects are excluded even if someone logged time there. Co-leads (e.g. Mayur / Hasan) share one roster. |
| **tasks** CSV | Time Doctor **daily tasks** style export: drives **which cards** appear (distinct **Project** + **Task**). **Task** must match the Master Tracker card **Name**. |
| **summary** CSV | Time Doctor export of **total tracked time per task (all developers)** — one row per **Project** + **Task**; used as **Hours Tracked** for each card. |

File names do not have to be `tasks.csv` / `summary.csv`; content and columns must match what the skill describes (see **Exports** below).

## Exports (Time Doctor)

- **Tasks file** — daily (or day-scoped) **projects and tasks** report with columns such as **Project**, **Task**, **User**, **Date**, **Time Tracked (Hours & minutes)**, etc. The agent uses it to decide which **(Project, Task)** cards to include.
- **Summary file** — report whose columns are exactly **Project**, **Task**, **Time Tracked (Hours & minutes)** with **rolled-up** totals per task (all devs). Used for **Hours Tracked** on each line.

Rows where **Project** is **EcomExperts** are ignored.

## Prerequisites

- **Notion MCP** enabled and authenticated so the agent can search/fetch Master Tracker cards and, when needed, read **comments** on the **master map card** for Notes.
- Master Tracker cards whose **Name** matches the **Task** column in your CSV (and **Project** / client context when there are duplicates).

## Output shape (typical)

By default you get a **single copy-paste block** (markdown `` ```text `` fence) suitable for Slack: title line **Hour Tracking Report (DD/MM/YYYY)**, then **one account header per client**, all cards under that client grouped in **one** blockquote (`>` lines), with **Hours Estimated** (from Notion task **Duration** sums), **Hours Tracked** (from summary CSV), **Hours Task pushed to QI/Done**, and **Notes** from master-card comments when the date matches **`UPDATE (DD-MM-YYYY)`**.

Lead-filtered runs also append accounts on the roster that had **no** tasks that day as **`>No tasks for now.`**

## Variations

- **No lead name** — full batch from the CSV pairs (still drops EcomExperts); no roster tail unless you ask for it.
- **Single card only** — paste **one** Notion master card URL or title **without** attaching both CSVs; workflow is different (detail / optional Slack). Tracked hours may be omitted without summary CSV.
- **“Only my hours”** — say explicitly if each line should use **your** Time Doctor rows only; the agent sums **tasks.csv** instead of **summary.csv** for that line.
- **No notes** — say **skip notes** / **no notes** to avoid comment fetches.

Date phrases and roster names are defined in detail in **`SKILL.md`** (including supported lead names and aliases).

## Full specification

For agents and maintainers: behavior, filters, Slack layouts, comment matching, and roster tables live in **`SKILL.md`** in this folder.
