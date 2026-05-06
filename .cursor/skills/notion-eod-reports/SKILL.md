---
name: notion-eod-reports
description: >-
  Builds Master Tracker EOD reports from Notion plus tasks.csv and summary.csv: hours
  estimated from task Duration sums, hours tracked from CSV, QI/Done bucket sums, and
  clean copy-paste blocks without caveats. Also supports single-card breakdowns by Notion
  URL or title. Use when the user mentions Notion EOD reports, attaches
  tasks.csv and summary.csv, says follow notion EOD reports, generate my reports, PF cards,
  Print Fresh, hours estimated vs tracked, total tracked time per task (all developers), or
  Kanban Quality Inspection / Done totals, Slack format, Hour Tracking Report, copy-paste,
  single code block for chat, easy copy, as of a date, or Notes from Notion comments matching
  UPDATE (DD-MM-YYYY) on the master card only. Strict account grouping: one account header per
  distinct Project; all cards for that client listed under it—never repeat *Account:* per card.
  Lead-filtered batch when the user says generate EOD for a named lead (e.g. for Mayur) plus CSVs:
  only accounts on that lead's roster; exclude other Time Doctor projects even if the same dev logged time there.
  Under each account, all card lines sit in one Slack blockquote (prefix each line with >); lead-filtered runs also list
  roster accounts with zero tasks at the end (> No tasks for now.).
---

# Notion EOD reports

**Notion data must come from MCP tool results only — never guess.** For single-card task tables, follow **[Single-card workflow (URL or title only)](#single-card-workflow-url-or-title-only)** below.

**Never use** the Master Tracker page field **`Actual dev hours used`** (or similar Time Doctor totals) as **Hours Estimated**. Estimated hours are always the **SUM(`Duration`)** from **All Tasks [EE]** rows linked via the master card’s **`Tasks`** relation, after the filters below.

### Account grouping (strict)

For **batch** and **any multi-card** output (Slack easy copy default, Slack rendered blockquotes, strict plaintext, legacy fenced layouts):

- Emit **exactly one** account header per distinct **`Project`** (same **`AccountDisplay`** after normalization): e.g. a single `*Account:* \`Printfresh\`` or a single strict `Account: Printfresh` or one `` `Account: Printfresh` `` chip for rendered Slack—**not** one account line per card.
- Under that header, list **every** card for that client in sort order: each card is **only** `*Card:*` … hour lines … then **`Notes:`** with **`•`** bullets on separate lines when applicable (or the format’s equivalent field lines). **Do not** prefix each card with another `*Account:*` / `Account:` / account chip for the same Project.
- **Slack / default batch layout:** Keep the **account header** (`*Account:*` / account chip) **outside** the blockquote; put **all** card field lines for that account **inside one blockquote** by prefixing each of those lines with **`>`** (see [Slack — easy copy (default)](#slack--easy-copy-default)). Between two cards under the same account, keep the quote continuous (e.g. a line that is only **`>`** or **`> `** if you need a visual gap).
- When **`Project`** changes, end the group (blank line as below), then emit **one** new account header for the next Project.

**Invalid:** two consecutive `*Account:* \`Printfresh\`` blocks for two Print Fresh cards. **Valid:** one `*Account:* \`Printfresh\`` (no `>`), then the first card’s field lines **each prefixed with `>`**, optional `>`-only separator line, then the second card’s **`>`**-prefixed lines (no second account line until a different client appears).

Single-card-only output is unchanged: **one** account line and **one** card block.

---

## Tools

Use the workspace Notion MCP (`notion-fetch`, `notion-search`, **`notion-get-comments`**, etc.). Read schemas before calling.

For **discussions / comments** on the **master map card** only: **`notion-get-comments`** with that card’s **`page_id`** returns full comment bodies (never pass a linked **Tasks** page id for Notes). **`notion-fetch`** with **`include_discussions: true`** is **optional**—use it only when you need **`discussion://`** anchors to debug a miss; otherwise skip it to avoid extra markup in context. If **`notion-get-comments`** returns a **transient server error**, retry **once** for that `page_id`; if it still fails, continue without Notes from comments and mention the failure only if the user asked for diagnostics.

### Token efficiency (keep runs cheap)

Most “millions of tokens” EOD runs come from **large Notion tool payloads** (many pages × long Markdown/XML) and **long chat history**, not from this skill file.

**Chat / inputs**

- Use a **fresh agent thread** per EOD: one short user message + CSVs + date + lead. Avoid carrying prior tool dumps or earlier reports in the same conversation.
- **Pre-filter CSVs** before the run (outside the agent if needed): **report day** + **lead roster** so the distinct **(Project, Task)** set is as small as possible. Do not paste unrelated exports.

**Notion fetch volume (batch)**

- **One** `notion-fetch` per **master** map card (per **Task**), plus **one** `notion-fetch` per **linked task page** in that card’s **`Tasks`** relation needed for sums—**no duplicate fetches** for the same `page_id` in one run.
- Do **not** fetch task pages for cards you will not emit (already filtered by CSV rules).

**Notes (`notion-get-comments`) — tiered (default)**

Many threads live on the **page**; pulling **every block’s** discussions first is often huge. Use **at most two** calls per master **`page_id`** when Notes are in scope:

1. **Pass A (default):** `include_all_blocks: false`, **`include_resolved: true`** so resolved threads still count. Scan XML for **`UPDATE (`** + **`DD-MM-YYYY`** + **`)`**.
2. **Pass B (only if Pass A found no match and the deliverable still expects Notes):** `include_all_blocks: true`, **`include_resolved: true`**.

If the user says **no notes** / **skip notes**, **do not** call **`notion-get-comments`** at all (and you may omit **`include_discussions`** on fetches).

**Final reply**

- Output **only** the report (fenced Slack block, etc.); **do not** paste raw Notion XML, full MCP payloads, or long tables of intermediate properties into the user-visible answer.

---

## CSV inputs (batch EOD)

The user supplies two files (typically named **`tasks.csv`** and **`summary.csv`**).

### tasks.csv — daily tasks export

Expected columns include: **Project**, **Task**, **User**, **Email**, **SAM Compatible Name**, **Date**, **Time Tracked (Hours & minutes)** (same shape as `projects-and-tasks-report-daily-*.csv`).

### summary.csv — total tracked time per task (all developers)

Columns are **exactly**:

| Column | Meaning |
|--------|--------|
| **Project** | Client / project group (e.g. `Print Fresh`) |
| **Task** | Time Doctor (or export) **task name**; must match the Master Tracker card **`Name`** for lookup |
| **Time Tracked (Hours & minutes)** | **Total** time logged on that **Task** for the period — **aggregated across all developers** (and any other dimensions the export rolled up). This is the authoritative value for **Hours Tracked** / “total hours tracked” for that card. |

There is **no** per-user row in this file: one row per **(Project, Task)** with the **rolled-up** total. Use it whenever the user needs a **correct org-wide** tracked total for a task, not a single person’s hours.

### CSV rules

1. **Ignore all rows** where **Project** equals **`EcomExperts`** (case-insensitive).
2. Use **tasks.csv** to decide **which cards** to include: distinct **(Project, Task)** pairs from non-ignored rows. The **Task** value is the Master Tracker card **`Name`** (e.g. `PF - SSO`).
3. Use **summary.csv** for **Hours Tracked**: find the single row with the same **Project** and **Task**, read **Time Tracked (Hours & minutes)**. That value is already the **total tracked** for that task (all devs). Convert to a readable phrase (e.g. `17h 01m` → `17 hours 1 minute`). If the pair is missing, **Hours Tracked** is unavailable for that card — still without long explanations in brackets.

**Lead-filtered runs** (see [Lead-filtered batch](#lead-filtered-batch)): after step 1, **restrict** rows to those whose **Project** matches an account on the resolved lead’s **[Lead account roster](#lead-account-roster)** (case-insensitive + alias rules there). Then take distinct **(Project, Task)** only from those rows. **Hours Tracked** stays from **summary.csv** for each included pair (time **on that client’s task**—any developer). **Do not** include cards whose **Project** is outside that roster (e.g. **Pet Planet** does not appear on **Mayur**’s report even if someone logged time there the same day).

---

## Trigger phrase

When the user attaches **`tasks.csv`** and **`summary.csv`** and says something like **follow notion EOD reports** or **generate my reports**, run the **Batch EOD** workflow below.

When they also name a **lead** (phrases like **for Mayur**, **for lead Mayur**, **Mayur’s accounts**, **EOD for Hasan**), treat the run as **[Lead-filtered batch](#lead-filtered-batch)** after resolving the lead and roster.

When they also give a **report day** (e.g. **as of 29-04-2026**, **as of 2026-04-29**, **EOD April 29 2026**), parse it to canonical **`YYYY-MM-DD`** and use **[Notes from Notion comments (report date)](#notes-from-notion-comments-report-date)** when the deliverable should include **Notes** (print those lines **only** when note content exists per that section). For Slack / copy-paste batch output, the report title date uses **`DD/MM/YYYY`** derived from that same calendar day (see [Slack — easy copy (default)](#slack--easy-copy-default)).

When a **report day** is given, prefer **tasks.csv** rows whose **Date** column parses to that same calendar day when building the distinct **(Project, Task)** set (so the batch reflects **that day’s** work). If **Date** is missing or the export is already scoped to one day, use all non-ignored rows. **Lead-filtered:** apply the date filter **before** applying the roster filter so empty lead reports mean “no qualifying rows that day,” not “wrong roster.”

---

## Lead account roster

These lists mirror the workspace **Lead developer → client account** boards. **Time Doctor `Project`** values in CSVs should match one of the **Match as** strings (case-insensitive); treat commas in this doc as separate aliases for the same account.

**Maintenance:** When accounts move between leads or new clients are added, update this section so the skill stays accurate. Do **not** invent accounts for a lead.

### Roster — Mayur and Hasan

`Print Fresh`, `Printfresh`, `Print fresh` · `Boundless`, `Carvana` · `Golden Lions` · `She's Birdie` · `My Medic`, `MyMedic` · `Wingstop` · `Hera Fine Jewelry`, `HeraFineJewelry`

### Roster — Nitin and LJ

`Jones Road Beauty`, `Jones Road` · `Taos Footwear`, `Taos` · `Voluspa`

### Roster — Anshuman and Mae

`FireWarehouse`, `Firewarehouse` · `FireResQ`, `FireResq`, `FireResq Bulk Hours` · `Trina Turk` · `The Swank Company`, `TheSwankCompany` · `Svaha` · `Wild Roman` · `Caddis` · `eHouse`, `E-house`, `E-house/blkhrs` · `La Canadienne`, `La Canadienne/blkhrs`

### Roster — Khaled and Francheska

`The Treadmill Factory`, `Treadmill Factory` · `Fatty15` · `WonderFold`, `Wonderfold` · `Warroad`

### Roster — Mujtaba and JR

`Create Room` · `Legends` · `Growthday` · `Northern Fitness` · `Tucker Carlson Network` · `HockeyStickMan`

### Roster — Moosa and Rosie

`Pet Planet` · `Malbon Golf - Buckets Club`, `Malbon Golf` · `Mountain House` · `Mind Body Green` · `Shepherds Fashion` · `Made By Mary`

### Roster — Rubby and Angie

`Tailwind Nutrition` · `JD Sports Canada`, `JD Sports` · `Choczero` · `Welleco` · `TruHeight Vitamins`

### Roster — Mohannad and John

`Seeking Health` · `Bruce Bolt` · `Supergut` · `Voomi Supply`

### Lead name → roster (resolution)

Match the user’s **lead token** (e.g. **Mayur**, **mayur**, **Hasan**, **JR**, **Moosa**) **case-insensitively** to **any** first name in a column header below; use the roster for that **whole column** (shared accounts for co-leads).

| If the user says (contains) | Use roster section |
|-----------------------------|----------------------|
| Mayur, Hasan | [Roster — Mayur and Hasan](#roster--mayur-and-hasan) |
| Nitin, LJ | [Roster — Nitin and LJ](#roster--nitin-and-lj) |
| Anshuman, Mae | [Roster — Anshuman and Mae](#roster--anshuman-and-mae) |
| Khaled, Francheska, Francesca | [Roster — Khaled and Francheska](#roster--khaled-and-francheska) |
| Mujtaba, JR | [Roster — Mujtaba and JR](#roster--mujtaba-and-jr) |
| Moosa, Rosie | [Roster — Moosa and Rosie](#roster--moosa-and-rosie) |
| Rubby, Angie | [Roster — Rubby and Angie](#roster--rubby-and-angie) |
| Mohannad, John | [Roster — Mohannad and John](#roster--mohannad-and-john) |

If the token matches **no** row (e.g. typo): **do not** guess a roster—say the lead is unknown and ask for a name from the table, or run an **unfiltered** batch only if the user explicitly asks to proceed without lead filtering.

---

## Lead-filtered batch

Use this when the user asks for an EOD for a **named lead** plus **`tasks.csv`** and **`summary.csv`**.

1. **Parse** the **report day** if present (same as [Report date parsing](#report-date-parsing-when-the-user-specifies-a-day)).
2. **Resolve lead:** From the user message, take the name after **for** / **for lead** / similar (e.g. **…generate … for Mayur** → **Mayur**). Map with **[Lead name → roster (resolution)](#lead-name--roster-resolution)**.
3. **Build allowlist:** Collect every **Match as** string for that roster entry into a set. A **tasks.csv** row’s **Project** is **allowed** if `normalize(Project)` equals `normalize(alias)` for any alias, where **normalize** means: trim ASCII spaces, Unicode normalize, **casefold**, collapse internal runs of spaces to one.
4. **Filter tasks.csv:** Drop **EcomExperts**; drop rows whose **Project** is not allowed; when a report day is set, keep rows whose **Date** parses to that day (if the column exists and parses; otherwise keep all rows and rely on export scope).
5. **Card set:** Distinct **(Project, Task)** from the filtered rows. **Omit** any **Project** not on the allowlist (e.g. another dev’s **Pet Planet** time never creates a Pet Planet card on **Mayur**’s report). Work logged under **Print Fresh** still yields **Print Fresh** cards when **Print Fresh** is on that lead’s roster.
6. **Notion + output:** For each pair, run the same steps as **[Batch EOD workflow](#batch-eod-workflow)** (master card, Status, Tasks relation sums, etc.). **Hours Tracked:** from **summary.csv** for that **Project** + **Task** (time on **that task / client**, all developers)—same as non-lead batch for portfolio visibility under that lead’s accounts.
7. **Optional “only this person’s hours”:** If the user explicitly asks for **only {Person}’s hours** / **my Time Doctor only** on each line, compute **Hours Tracked** by summing **tasks.csv** **Time Tracked** for rows where **User** or **Email** matches that person (case-insensitive) **and** **Project** is allowed **and** **Task** matches **and** (if applicable) **Date** matches the report day—**do not** use **summary.csv** for that line.
8. **Empty roster accounts (tail):** Treat each **`·`-separated segment** in that lead’s roster row as **one logical client** (comma-separated names inside a segment are **aliases** of the same client—e.g. `Boundless`, `Carvana` is **one** slot). After emitting every account group that has **at least one** card (sorted as in the batch workflow), append **one block per logical client that had zero cards** this run: emit **one** `*Account:* \`{AccountDisplay}\`` line (same normalization as accounts with work—pick a single display string per segment, e.g. first alias after normalization or the canonical chip like **`MyMedic`**, **`HeraFineJewelry`**, **`She's Birdie`**), then a **blockquote** whose only content line is **`>No tasks for now.`** (exact sentence; no extra label). **Sort** these empty slots by **`AccountDisplay`** (case-insensitive) and place them **after** all accounts that had cards. **Non-lead** batch runs: **omit** this tail unless the user explicitly asks to list a roster.

---

## Batch EOD workflow

For **each** distinct **(Project, Task)** from **tasks.csv** after the EcomExperts filter **and** after any **[Lead-filtered batch](#lead-filtered-batch)** roster + date filters:

1. **Locate the master card** in Notion: **`notion-search`** / **`notion-fetch`** so the page is a **Master Tracker [EE]** map card whose **`Name`** matches **Task**. Prefer the correct client context if duplicates exist (match **Project** / client when needed).
2. **Main card `Status`:** read from master properties. For the report line **Status**, present plain wording (e.g. strip leading emoji; `🏗In Progress` → `In progress`).
3. **`Tasks` relation:** fetch every linked task page. For each task read **`Title`**, **`Duration`**, **`Kanban Status`**, **`Parent Task`** (if present), and whether the page is **deleted** / excluded per filters.
4. **Filters (always this order):**
   - Drop **deleted** tasks from all sums and lists.
   - **Top-level only:** **`Parent Task`** empty / absent.
   - Exclude **`Title`** containing **`build header`** (case-insensitive).
5. **Hours Estimated:** SUM(**`Duration`**) over included tasks; missing **Duration** counts as **0** for the sum. Express as a number with decimal when needed (e.g. **17.6 hours**).
6. **Hours Task pushed to QI/Done:** SUM(**`Duration`**) over included tasks whose **`Kanban Status`** is exactly **`🔍Quality Inspection`** or **`✅Done`** (use the workspace’s actual select strings from MCP).
7. **Hours Tracked:** from **summary.csv** for that **Project** + **Task** (one row; **total** time on that task for all developers), formatted readably. **Exception:** if the user asked for **only {Person}’s hours** / **my Time Doctor only** under **[Lead-filtered batch](#lead-filtered-batch)**, use the **tasks.csv** sum described there instead of **summary.csv** for that card.
8. **Account label:** derive **`AccountDisplay`** from **Project** for grouping and display (see **`AccountDisplay` normalization** under [Slack — easy copy (default)](#slack--easy-copy-default); strict plaintext may use a readable `Account: …` line without the Slack chip rules). Used **once per account group** (see grouping below).
9. **Order and grouping:** After computing all cards, **sort** by **`Project`** (CSV value, case-insensitive), then by **`Task` / card Name**. **Group** every card that shares the same **`Project`** (case-insensitive match on the CSV **Project** string). Emit **one** account header **per distinct Project** (strict: `Account:` line; Slack: follow **[Slack — easy copy (default)](#slack--easy-copy-default)** or **[Slack rendered — blockquotes](#slack-rendered--blockquotes-no-outer-fence)**), never repeated for each card under that client — see **[Account grouping (strict)](#account-grouping-strict)**.
10. **Notes from comments (when a report date is given):** If the user specified a **report day** (e.g. **as of 29-04-2026**) and the output needs **Notes** (Slack or strict), for **each** report row call **`notion-get-comments`** **only** on that row’s **master Master Tracker [EE] map card** `page_id` — see **[Notes from Notion comments (report date)](#notes-from-notion-comments-report-date)**. Do **not** pull comments from linked **Tasks** pages, subtasks, or any **other** master card.

### Grouping by account (required for batch)

The **same** grouping rule applies whether there is **one** card or **many** for a **`Project`**: **never** repeat the account header for each card—**one** account line per distinct Project, then all its cards underneath ([Account grouping (strict)](#account-grouping-strict)).

When **two or more** cards share the same **`Project`** (e.g. both `Print Fresh`):

- **Strict plaintext:** Print **`Account: …` once** for that group, then print **each card’s** `Card:` through `Hours Task pushed to QI/Done:` lines in a row. Put a **blank line** between card blocks **inside** the same account. Put a **blank line** before the next **`Account:`** when the Project changes.
- **Slack / EOD / copy-paste (default):** Use **[Slack — easy copy (default)](#slack--easy-copy-default)** (account line unquoted, **all** card lines under it **`>`**-prefixed in one blockquote) unless they ask for **strict plaintext only**, **rendered blockquotes only** (then use [Slack rendered — blockquotes](#slack-rendered--blockquotes-no-outer-fence)), or a named legacy layout.

Single-card batches are unchanged: one account line, one card block.

### Report date parsing (when the user specifies a day)

Phrases like **as of 29-04-2026**, **29/04/2026**, **2026-04-29**, **April 29, 2026**, or **4/29/2026** must be parsed to a single calendar day. Normalize internally to **`YYYY-MM-DD`**.

**Notion comment matching (strict):** Notes from master-card comments use **only** this pattern. From the parsed calendar day, build **`DD-MM-YYYY`** with **two-digit** day and month (zero-padded) and **four-digit** year (e.g. 2026-04-29 → **`29-04-2026`**). A comment qualifies **only** if its body contains the **exact ASCII substring** **`UPDATE (`** + **`DD-MM-YYYY`** + **`)`** — the word **`UPDATE`** must match **case** as shown (uppercase). Examples that **count**: `UPDATE (29-04-2026)`, `**UPDATE (29-04-2026):**` after stripping markup still contains the substring. Examples that **do not** count (ignore for Notes, even if the same calendar day appears elsewhere): `4/29/2026`, `2026-04-29`, `April 29, 2026`, `QI … 4/29/2026`, or any line that never includes **`UPDATE (`**…**`)`** for that **`DD-MM-YYYY`**.

Rough strip for **detection only:** treat `<br>` as newline/space and remove simple angle-bracket tags so `**UPDATE (29-04-2026):**` is searchable; the rule is still **substring** **`UPDATE (DD-MM-YYYY)`**, not loose date matching.

### Notes from Notion comments (report date)

When a **report date** is present and the user asked for output that **may** include **Notes** (Slack `*Notes:*` or strict **`Notes:`** when they want dated notes on strict lines):

**Source (strict):** **`notion-get-comments`** is called **only** on the **master Master Tracker [EE] map card** for the card being reported — the page whose **`Name`** matches **Task** / the URL the user opened (e.g. **PF - International Shipping Bar Toggle**). **Do not** call **`notion-get-comments`** on:

- Any page in the **`Tasks`** relation (**All Tasks [EE]** rows),
- Subtasks or nested task pages,
- Any **other** master Tracker card (only the current row’s master `page_id`).

Use the **tiered** **`notion-get-comments`** strategy in **[Token efficiency](#token-efficiency-keep-runs-cheap)** (Pass A, then Pass B only if needed). Block-anchored threads on **that same master page** are included only after Pass B; they are **not** “other cards.”

1. Call **`notion-get-comments`** with **`page_id`** = that **master map card** UUID (from **`notion-fetch`** on the same page), following **Pass A** / **Pass B** as in [Token efficiency](#token-efficiency-keep-runs-cheap).
2. From the returned XML, collect every `<comment … datetime="…">` body in **all** discussions returned for that **single** `page_id`.
3. Keep comments whose body contains the **exact substring** **`UPDATE (`** + **`DD-MM-YYYY`** + **`)`** for the report day (see [Report date parsing](#report-date-parsing-when-the-user-specifies-a-day)). **Ignore** all other date mentions.
4. If **more than one** comment matches, choose the one with the **latest** `datetime` (ISO on the comment node).
5. **Compose Notes — cleanup:** Use the **full** matched comment body (or from the **`UPDATE (`** line downward if you trim only for readability). Replace `<br>` with newline or space; strip angle-bracket HTML/XML tags; **do not invent** content. Remove decorative bold markers when they only wrap headings; **do not** add **`(...)`** or other bracketed asides. If the comment is mostly links, keep **at most one** plain URL or drop URLs and keep the sentence—never empty `()` / `[]`. Optionally trim very long QI dumps (~400 chars / first paragraph); keep the opening sentence. **Do not** repeat the literal **`UPDATE (DD-MM-YYYY)`** heading as its own bullet unless it still reads naturally—prefer the substantive lines **after** it.

6. **Compose Notes — bullet structure (required):** Never emit Notes as **one** long line joined by **`;`**. After cleanup, build **one bullet per line** using a leading **`•`** (bullet U+2022) followed by **one space**, then the clause text.

   - **Semicolon lists:** If the prose contains **`;`** between clauses (with or without a trailing space), **split** on **`;`**. Trim each segment; each non-empty segment becomes **its own** bullet line.
   - **Already multi-line:** If newlines already separate distinct points, treat each non-empty line as one bullet (strip any leading `-`, `*`, or `•` from the source first).
   - **Single sentence:** If there is **no** `;` and effectively one short sentence, output **one** bullet line (still use **`• `**, not a naked paragraph).
   - **Capitalization:** Preserve meaning; trim trailing semicolons/spaces on each bullet.

7. **No match:** **Omit** the entire **Notes** block for that card (**no** lone **`Notes:`** line with excuses). **Do not** substitute Notes from **Status** + CSV.

**Important:** A comment’s **`datetime`** alone never qualifies it; the **body** must contain **`UPDATE (DD-MM-YYYY)`** for the report day. Example: **PF - International Shipping Bar Toggle** had **`UPDATE (29-04-2026):`** on the master thread; QI lines with **`4/29/2026`** without that **`UPDATE (`**…**`)`** substring are **ignored** for Notes.

### Batch output format — strict

Emit **only** labeled lines per card. **Do not** add explanations in parentheses, footnotes, “per skill”, source citations, or correction notes after values.

Use this shape. **One `Account:` per `Project` group** (never repeat `Account:` for a second card on the same client); for **each** card repeat the hour lines; after those lines add **`Notes:`** + **`• …`** bullets **only** when Notes apply (see below).

```text
Account: …

Card: …
Status: …
Hours Estimated: …
Hours Tracked: …
Hours Task pushed to QI/Done: …
Notes:
• …
• …

Card: …
Status: …
Hours Estimated: …
Hours Tracked: …
Hours Task pushed to QI/Done: …
```

If only one card exists for that Project, there is still a single `Account:` line and one card block (blank line optional). No markdown tables unless the user explicitly asks for task-level tables.

When the user wants **Notes** on strict lines too: after the core hour lines for that card, print **`Notes:`** on its **own** line, then **each bullet on its own line** as **`• `** + clause (same [Compose Notes — bullet structure](#notes-from-notion-comments-report-date)) — **never** one semicolon-joined line. If there is nothing to show, **omit** the entire **Notes** block (same five core lines only).

### Slack — easy copy (default)

When the user asks for **Slack**, **EOD**, **copy-paste**, **Hour Tracking Report**, **easy copy**, or batch output **without** **[strict plaintext](#batch-output-format--strict)** only, the assistant message must include **one** markdown **code fence** so the whole report is a **single copy region** in chat (Cursor, Slack, etc.—same pattern as a Slack “Copy” code block).

**Outer fence (required):**

1. Open with a line that is **only** **` ```text`** (three backticks + the word `text`). Nothing else on that line.
2. Put **the entire report** on the following lines — title through the last line of the last card **or** the last **empty-roster** block (`>No tasks for now.`) on lead-filtered runs. **No** commentary, apologies, or “here is your report” **inside** the fence.
3. Close with a line that is **only** **` ``` `** (three backticks).
4. Optionally **one** short line **after** the closing fence is allowed (e.g. “Paste the fenced block into Slack for one code block with Copy.”). Do **not** duplicate the report outside the fence unless the user explicitly asked for a **second** format (see [Slack rendered — blockquotes](#slack-rendered--blockquotes-no-outer-fence)).

**Why:** The user copies once from the chat UI; wrapping in ` ```text ` … ` ``` ` matches the internal “single gray block + Copy” workflow. **Slack blockquotes:** lines inside the fence that start with **`>`** render as Slack quotes only after the user pastes the inner text **as a normal message** (not inside Slack’s own code block). If they paste the whole fenced snippet into Slack as a code block, quotes stay literal—remind them in the optional post-fence line when helpful.

**Inner format (default — account header + blockquoted cards):**  
Inside the fence, the **title** line has **no** `>` prefix. For each account, put **`*Account:* \`…\`` on its own line with **no** `>`, then put **every** card field line for that account **inside one Slack blockquote** by prefixing each line with **`>`** (space after `>` optional but consistent). When the user pastes the copied text into Slack **as a normal message** (not inside a code block), the gray bar groups all cards under that account.

- **Title:** `👉 Hour Tracking Report (DD/MM/YYYY):` on the first inner line (plain text; no requirement to bold inside the fence). Use **zero-padded** **`DD/MM/YYYY`** for the report day (or today if unspecified). **No** `>` on this line.
- **Blank line**, then for **each distinct `Project` / `AccountDisplay` group** that has **at least one** card (sorted as in the batch workflow):
  - **Once per group (not blockquoted):** `*Account:* \`{AccountDisplay}\`` — Slack-style bold label + **backticks only around the account name** (not the words `Account:`), e.g. `` *Account:* `Printfresh` `` per normalization below. **Do not** repeat this line before every card in the same group.
  - **Immediately below**, for **each card** under that account (in **`Task` / Name** sort order within the group), emit these lines **each prefixed with `>`**:
    - `>*Card:* [{master Name}]({master URL from notion-fetch})`
    - `>*Status:* {plain status}` — strip emoji; **Title Case** is acceptable (e.g. `In Progress`, `Ready For Client`) or **sentence case**; stay consistent in one report.
    - `>*Hours Estimated:* {compact duration}` — use **`Xhrs`** or **`Xhrs Ymin`** (convert from decimal hours: e.g. 17.6 → `17hrs 36min`; whole hours → `105hrs`). Do **not** use only bare decimals in this default inner format.
    - `>*Hours Tracked:* {same compact style}` from **summary.csv** by default (convert `46h 08m` → `46hrs 8min`, etc.). If the run used **only this person’s hours** ([Lead-filtered batch](#lead-filtered-batch)), use the summed **tasks.csv** value instead.
    - `>*Hours Task pushed to QI/Done:* {same compact style}` from the QI/Done **Duration** sum.
    - **Notes (when present):** **`>*Notes:*`** on its **own** quoted line (label only, no body); then **one quoted line per bullet**: **`>• {clause}`** — same **`•`** + space as [Compose Notes — bullet structure](#notes-from-notion-comments-report-date). **Never** join bullets with **`;`** on a single **`>*Notes:*`** line. Omit all Notes lines if no match (same rules as [Notes from Notion comments](#notes-from-notion-comments-report-date)).
  - **Between two cards** under the **same** account: keep one continuous blockquote—use a **`>`**-only line (or `> `) between card blocks if you need separation without breaking Slack’s quote run.
- When **Project** changes: after the last quoted line of the previous group, use a **blank line** (no `>`), then the next **`*Account:*`** line (no `>`), then its quoted card lines.
- **[Lead-filtered batch](#lead-filtered-batch) only — empty roster tail:** After all accounts that had cards, output **blank line**, then for each **logical roster client** with **zero** cards this run (step **8** in lead-filtered): **`*Account:* \`{AccountDisplay}\``** (not quoted), then **one** quoted line **`>No tasks for now.`** Sort these empty slots by **`AccountDisplay`** (case-insensitive). See [Lead-filtered batch](#lead-filtered-batch) step 8.

**`AccountDisplay` normalization** (same as before): e.g. `Print Fresh` → **`Printfresh`**, `Hera Fine Jewelry` → **`HeraFineJewelry`**, `Pet Planet` → **`PetPlanet`**, `Wingstop` → **`Wingstop`**, unless CSV already uses a single token—stay consistent across runs.

**Notes sourcing:** unchanged — **[Notes from Notion comments (report date)](#notes-from-notion-comments-report-date)** when a report day applies.

**Minimal example (documentation only — real runs must include the outer ` ```text ` fence):**

```text
👉 Hour Tracking Report (29/04/2026):

*Account:* `Printfresh`
>*Card:* [PF - International Shipping Bar Toggle](https://app.notion.com/p/…)
>*Status:* Ready For Client
>*Hours Estimated:* 4hrs 57min
>*Hours Tracked:* 4hrs 50min
>*Hours Task pushed to QI/Done:* 4hrs 57min
>*Notes:*
>• Country code display is covered in existing PF - Trello UA Tasks work
>• announcement bar for international customers is covered in PF - International Shipping Bar Toggle
>• AU free shipping tested and working
>• card left on Need Support for these confirmations
>
>*Card:* [PF - SSO](https://app.notion.com/p/…)
>*Status:* In Progress
>*Hours Estimated:* 17hrs 36min
>*Hours Tracked:* 17hrs 1min
>*Hours Task pushed to QI/Done:* 14hrs

*Account:* `MyMedic`
>No tasks for now.
```

**Native Slack links:** If the user explicitly asks, use `*Card:* <https://…|Card title>` instead of markdown `[text](url)` **inside** the same fenced block.

---

### Slack rendered — blockquotes (no outer fence)

Use **only** when the user explicitly asks for **rendered** Slack, **blockquote**, **gray bar**, or **no code block** / **not in a fence**.

- **Do not** wrap this variant in ` ``` ` — it is meant to be pasted as normal Slack message text so `>`, links, and italics render.
- **Title:** `👉 Hour Tracking Report (DD/MM/YYYY):` then blank line.
- **Per distinct Project:** **one** account chip `` `Account: {AccountDisplay}` `` on its own line (not blockquoted)—same grouping rule as [Account grouping (strict)](#account-grouping-strict); **do not** repeat the chip before every card for the same client.
- **Each card** under that account: prefix **every** line of that card with **`>`**. Use **plain** labels **without** Slack bold on the keys (e.g. `>Card:`, `>Status:`, `>Hours Estimated:`, …) so the layout matches the “quoted / italic body” internal style; the card name stays a markdown **`[Title](url)`** link on the `Card:` line.
- **Hours:** `>Hours Estimated: {N.NN} hrs` and `>Hours Task pushed to QI/Done: {N.NN} hr` (decimal + `hrs` / `hr`). **Hours Tracked:** `>Hours Tracked: N Hour M Minutes` or `>Hours Tracked: Nh` when minutes are zero.
- **Notes:** when present: **`>Notes:`** on its own quoted line, then **`>• …`** on separate quoted lines — **never** `>Notes: • … • …` on one line. Omit entirely if no content.

**Between cards / accounts:** blank line between card groups under the **same** account; when moving to the **next** Project, blank line then **one** new `` `Account: …` `` line (never duplicate the chip between cards of the same client).

**Lead-filtered — empty roster tail:** Same as [Slack — easy copy (default)](#slack--easy-copy-default): after all accounts that had cards, append each **zero-card** logical roster client with **one** account chip line (not quoted) and **`>No tasks for now.`** — sorted by **`AccountDisplay`**.

---

### Legacy / alternate labels

If the user asks for **no header line**, **decimal hours** inside the **fenced** block instead of `Xhrs Ymin`, or a **different label set**, honor that inside the **same** single ` ```text ` wrapper unless they asked for strict plaintext only. **[Account grouping (strict)](#account-grouping-strict)** still applies: **one** account header per Project, **all** card field lines under it still **`>`**-prefixed when using the default Slack fenced layout.

---

## Single-card workflow (URL or title only)

When the user gives **one** Notion URL or **one** master card title (no CSV batch):

1. Fetch the **Master Tracker** master page.
2. Apply **`Tasks`** relation → per-task fetch → **filters** above.
3. Output may use this **detailed table** format: columns **Task name** | **Duration** | **Kanban Status (task)** | **Main card Status**, then **SUM(Duration):**, optional **Skipped (deleted only):**, **Errors** if needed.
4. If they also pass a **report date** and want **Notes** or **Slack / copy-paste** output, run **[Notes from Notion comments](#notes-from-notion-comments-report-date)** for **that master map card only** (same `page_id` as step 1)—do **not** pull comments from **Tasks** / subtasks / other cards. For Slack, use **[Slack — easy copy (default)](#slack--easy-copy-default)** (one account line, then **`>`**-prefixed card lines inside the single ` ```text ` fence; **Hours Tracked** may be omitted or marked unavailable if no CSV was supplied).

---

## Errors

If a task page in the relation cannot be fetched, note it once under **Errors** and continue (single-card flow). For batch EOD, keep the card block clean; mention fetch failures briefly only if the user needs them.
