---
name: agenthog-report
description: Write, publish and keep current a shared AgentHog report — a Markdown file with live charts that everyone in the organization opens in the dashboard. Learn the house format from an existing report, decide what is analysis (prose, frozen rows) and what stays live (constrained SQL with a TTL), preview until clean, publish with the ah CLI, verify by reading the resolved report back, and revise it on a schedule so its history stays intact. Use when asked for a report, a weekly/monthly review, a launch or liveops page, a KPI page, or to update or refresh an existing AgentHog report.
when_to_use: The user asks to "write up this week's numbers", "make an AgentHog report", "publish these numbers as a report the team can open", "build a launch/liveops page in AgentHog", "update the weekly report", "why is that report widget showing an error", or names an AgentHog report slug or `ah reports`. For installing AgentHog use agenthog-integrate; for A/B tests use agenthog-experiment.
---

# Write a shared report with AgentHog

A report is one Markdown file. Prose is your analysis; every top-level ```` ```widget ````
fence is a chart or table — either **frozen** rows you supply, or a **live** SQL query the
server re-runs within its TTL. Publishing it puts it at `https://agenthog.io/reports/<slug>`
for every member of the organization, at once (there is no draft state). Humans read there;
you write and revise it with the `ah` CLI. Every update is a new revision — nothing is ever
overwritten.

Follow the phases in order. Each ends with a check — do not proceed past a failed check.

## 0. Preconditions

- `ah` is installed and authenticated: `ah whoami` shows the `account` line — that is the
  organization the report will belong to. Drafting and `preview` work with **read** scope;
  `create`, `update`, `delete` need **write** scope (an org admin mints it at
  `https://agenthog.io/tokens`, or `ah login --write`). With several logins, every
  `ah reports` command needs `--account <org>`.
- You know which project(s) the report is about: `ah projects list` prints the keys
  (`ah_xxxxxxxx`). One report may draw on several.
- The data exists: `ah events top --since 7d` for the event names, `ah schema` for the
  tables and columns a live query can see. Never invent a column — check.

## 1. Learn the house format first

```
ah reports list                         # what the org already reads
ah reports read <slug>                  # the resolved report, exactly as a member sees it
ah reports pull <slug> --out prior.md   # the authoring file behind it
```

Match the structure, tone, section order and widget choices of the most recent report on the
same subject; readers compare weeks. If the org has none yet, `ah reports widgets` prints the
widget library with each widget's column contract and a working example.

## 2. Decide what is analysis and what stays live

- **Prose** is the point of the report: what happened, why, what to do. Date your claims in
  the text ("as of Aug 30") — the numbers next to them keep moving.
- **Frozen rows** (`"rows": [...]`) are for numbers that should stand still: the comparison
  you computed, targets, a snapshot the analysis refers to. Frozen widgets cost nothing to
  view.
- **Live queries** (`"sql": "..."`) are for what the reader should see current. Pick the TTL
  by how the reader watches, not by how fresh the data could be: a weekly review `"24h"`;
  something people glance at during the working day `"1h"` (the default); liveops during a
  launch `"5m"`–`"15m"`. A report holds at most 24 live widgets, and every live widget is a
  query — keep it to the numbers that matter.
- One report answers one question. Slug: lowercase letters, digits, hyphens; ≤ 64 chars.

## 3. Write the file

````markdown
---
title: Weekly growth
description: Where the funnel stands this week. Kept current by the growth agent.
project: ah_xxxxxxxx          # default project for widgets that name none
since: 7d                     # default window for live widgets (default 7d)
---

## Where we are

Signups are up 12% week over week (as of Aug 30), carried by the pricing page rewrite.
The live numbers below recompute on their own — the line under each one says when.

```widget
{ "widget": "stat", "title": "This week", "ttl": "1h",
  "sql": "SELECT count(DISTINCT s.anon_id) AS visitors, count(*) FILTER (WHERE e.name = 'signup') AS signups FROM events e JOIN sessions s ON s.id = e.session_id WHERE e.ts >= :since AND s.classification NOT IN ('crawler','suspected_bot','test')" }
```

```widget
{ "widget": "line", "title": "Signups per day", "ttl": "1h", "since": "30d",
  "sql": "SELECT date_trunc('day', e.ts)::date AS day, count(*) AS signups FROM events e JOIN sessions s ON s.id = e.session_id WHERE e.name = 'signup' AND e.ts >= :since AND s.classification NOT IN ('crawler','suspected_bot','test') GROUP BY 1 ORDER BY 1" }
```

```widget
{ "widget": "bar", "title": "Sessions by source", "width": "half", "ttl": "6h",
  "sql": "SELECT coalesce(utm_source, '(direct)') AS source, count(*) AS sessions FROM sessions s WHERE started_at >= :since AND s.classification NOT IN ('crawler','suspected_bot','test') GROUP BY 1 ORDER BY 2 DESC LIMIT 12" }
```

```widget
{ "widget": "table", "title": "Top events", "width": "half", "ttl": "1h",
  "sql": "SELECT e.name, count(*) AS n FROM events e JOIN sessions s ON s.id = e.session_id WHERE e.ts >= :since AND s.classification NOT IN ('crawler','suspected_bot','test') GROUP BY 1 ORDER BY 2 DESC LIMIT 20" }
```

## Against plan

```widget
{ "widget": "stat", "title": "Q3 targets",
  "rows": [ { "signups": 600, "conversion": 0.042 }, { "signups": 480, "conversion": 0.038 } ],
  "format": { "conversion": "percent" }, "labels": { "signups": "Q3 signups" } }
```

Next: turn on the retention email once the [experiment](/flags) reads.
````

Rules the parser enforces — and the reasons behind them:

- **Front matter** keys: `title` (required, or pass `--title`), `description`, `project`,
  `since`. Anything else is ignored with a warning.
- **A fence is strict JSON**: double quotes, no trailing commas, no comments. Only
  **top-level** fences are widgets — a fence inside a list or blockquote renders as a code
  block, and `preview` warns.
- **Exactly one of `rows` or `sql`.** Keys: `widget title note width rows sql project projects
  since until ttl x series stack format labels`. `format` per column is one of `number percent
  currency duration seconds text`; `labels` renames columns; `width: "half"` puts two
  widgets side by side, `width: "third"` three — good for a dense KPI band of stats.
- **Column contracts** — the shape your rows or query must produce:
  - `stat`: exactly **one row**; every column is a tile. An optional second row is the
    comparison and draws a delta badge. Aggregate without `GROUP BY`.
  - `line` / `area` / `bar`: an `x` column (default: the first column — a timestamp, date or
    text) plus one or more **numeric** columns as series (all of them, or the ones in
    `series`). `ORDER BY` the x column. `stack: true` stacks. Bars go horizontal when x is
    text with more than 8 categories.
  - `table`: any columns; add a `LIMIT` — up to 2,000 rows are kept.
- **Live SQL** runs under the same read-only, project-scoped guard as `ah sql`: one `SELECT`
  (or `WITH`), 10-second timeout, the tables `ah schema` lists. `:since` and `:until` are the
  widget's window: `since` comes from the widget, else the front matter, else `7d`; `until`
  only from the widget (default: now). Relative specs resolve **at compute time**, which is
  what makes a widget live. A literal date in the SQL is not live. `:until` may only appear
  when the widget sets `until`.
- **Exclude bots and test traffic in every live query** unless the report is about them:
  from `events`, `JOIN sessions s ON s.id = e.session_id` and
  `s.classification NOT IN ('crawler','suspected_bot','test')`; from `sessions`, the same
  `WHERE`. Live queries see raw rows — the default filter the other `ah` verbs apply is not
  applied for you. `preview` hints when a query touches events or sessions without it.
- **Column names are labels.** `count(*) AS signups`, not `count`. Ratios as fractions
  (`0.042`) with `"format": {"col": "percent"}`; money in major units with `currency`;
  seconds with `duration`.
- **Prose** is GFM (tables, task lists, footnotes, link references — they work across
  widgets). Raw HTML is dropped, not rendered. Links to dashboard pages may be relative
  (`/flags`).

## 4. Preview until clean

```
ah reports preview --file weekly.md          # validates, runs every widget once, saves nothing
ah reports preview --file weekly.md --full   # every row of every table
ah reports preview --file weekly.md --json   # the resolved blocks, for checking numbers
```

Two kinds of error, in two places:

- **Write-time** errors abort the command and name the line: `line 14: widget "stat":
  contract is exactly one row … — got 31 rows`, `line 20: unknown key "tittle" (did you mean
  "title"?)`. Fix and re-run until the file is accepted.
- **Compute-time** errors (a query that fails or returns the wrong shape) appear in the
  rendered output under the widget's `▸ Title  [kind · live]` line as
  `⚠ error at <time>: widget "<kind>": …` — the report still renders, that widget does not.

Then read the output as the reader will:

- every live widget shows rows and an `as of … · refreshes every …` footer, none shows a
  `⚠ error` line;
- a `hint:` about `classification` means a query is counting bots — add the filter, or state
  in the prose that bots are included on purpose;
- a `stat` tile that reads `0` or `—` usually means the wrong event name (`ah events top`).

## 5. Publish and verify

```
ah reports create weekly-growth --file weekly.md
```

It prints the URL and warms the cache so the first click is instant. The report is visible
to the whole organization from this moment. Then verify **from the published copy**, which is
exactly what a member sees:

```
ah reports read weekly-growth
```

Post `https://agenthog.io/reports/weekly-growth` with a one-line summary of the finding. Do not
paste the report into chat — the page has the live numbers, chat would have stale ones.

## 6. Keep it current

On the agreed cadence, revise the **same slug** — the history is the value:

```
ah reports pull weekly-growth --out weekly.md   # the current authoring file (--force to overwrite a differing local copy)
# revise the prose; keep widgets whose spec is unchanged (their cache carries over)
ah reports update weekly-growth --file weekly.md
ah reports read weekly-growth                   # verify
```

- `ah reports history <slug>` lists revisions; `ah reports diff <slug>` shows what the last
  update changed; `ah reports read <slug> --rev N` reads an older one. Create a new slug only
  when the reader wants separate per-period reports.
- A widget showing an error strip: `ah reports read <slug>` prints the full error text under
  the widget; `ah reports show <slug>` lists every widget's cache state (as of, fresh/stale,
  rows, the start of the last error) with its block id. Fix the query in the file and `update`.
- `ah reports refresh <slug> [--block <id>]` recomputes now (bounded to once per widget per
  60 seconds); the block ids are in `show`.
- `ah reports delete <slug> --yes` drops **every** revision. Only when the user asks.

## Errors you will meet

| message | fix |
|---|---|
| `widget fence is not valid JSON` | strict JSON: double quotes, no trailing commas, no comments |
| `unknown key "tittle" (did you mean "title"?)` | the key list in §3 |
| `give exactly one of "rows" (inline data) or "sql" (live query)` | a widget is frozen or live, never both |
| `widget "stat": contract is exactly one row … — got N rows` | aggregate without `GROUP BY`, or use `table` |
| `contract is an x column plus one or more numeric series columns` | cast counts to numbers; name the series; x is the first column |
| `series column "x" is not numeric in every row` | a `NULL`/text value slipped in — `coalesce(…, 0)` |
| `sql references :until but the widget sets no "until"` | add `"until"` or drop `:until` |
| `only read-only SELECT / WITH queries are allowed` | one statement, no writes |
| `query exceeded the 10000ms time limit` | narrow the window, aggregate more, add `LIMIT` |
| `project ah_… is not in this organization` | the key moved or is misspelt — `ah projects list` |
| `widget names no project and the organization has N — set \`project:\`` | the org has several projects: set `project:` in the front matter or on the widget |
| `N live widgets — the cap is 24` | freeze the ones that need not move as inline rows |
| `a widget fence inside a list renders as a code block` | move the fence to the top level |

## You are done when

- [ ] `ah reports read <slug>` shows the analysis with every live widget populated and no
      `error:` lines
- [ ] Every live query excludes `crawler`/`suspected_bot`/`test` sessions, or the prose says
      bots are included
- [ ] TTLs match how the reader watches the page (24h review · 1h daily · 5m–15m launch)
- [ ] The prose dates its claims and says what to do next
- [ ] You posted the report URL with a one-line summary, not the report body
- [ ] For an update: the same slug got a new revision (`ah reports history <slug>`), and
      `ah reports diff <slug>` shows only what you meant to change

Format reference and examples: `https://agenthog.io/docs/reports`.
