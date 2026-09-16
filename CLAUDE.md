# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-page Kanban board for UOB's internal IT PMO (demo/training tool, not a real UOB system). The entire app — markup, styles, and logic — lives in one file: `index.html`. There is no build step, no package manager, and no other source files.

## Commands

There is no build, lint, or test tooling in this repo. To run the app, open the file directly in a browser:

```bash
open "index.html"
```

There is no dev server, bundler, or test runner to invoke — changes are verified by reloading the file in a browser.

## Hard constraints (do not violate these when editing)

- **Vanilla HTML/CSS/JS only.** No frameworks (React, Vue, jQuery, Tailwind), no build step, no npm, no bundler. Everything must keep working via double-click / `file://`.
- **Single file.** All markup, the `<style>` block, and the `<script>` block stay in `index.html`. Do not split into separate `.css`/`.js` files.
- **No persistence.** Board state lives only in the in-memory `state` object. Do not introduce `localStorage`, `sessionStorage`, `IndexedDB`, cookies, or any storage API — a page refresh is expected to reset the board to the seeded demo data. The header note about this behavior should stay accurate if state handling changes.
- **No external resources.** No CDN scripts/styles, no Google Fonts, no image files. Icons are inline SVG or Unicode glyphs; fonts are the system font stack (`--font` custom property).
- **FormSubmit is the only backend call.** Task notifications POST to the FormSubmit AJAX JSON endpoint (`FORMSUBMIT_ENDPOINT` constant near the top of the `<script>` block). Never send the user's own email address anywhere except this one config constant, and never let a FormSubmit failure break board state — `notifyNewTask()` is always called from a `try/catch` and failures only surface as a non-blocking warning toast.

## Architecture

Everything reads from and writes to a single state object as the source of truth:

```js
const state = { tasks: [...], filters: { project, assignee, priority }, auditLog: [...] };
```

`renderBoard()` is the only place that rebuilds card DOM — it re-derives the four columns from `state.tasks` (via `applyFilters()`), calls `renderSummaryStrip()`, then calls `renderDashboard()` so the dashboard tab stays in sync with every mutation. No code should mutate card contents directly; instead mutate `state` and call `renderBoard()`.

Key functions (each does one job):
- `renderBoard()` / `renderCard(task)` / `renderSummaryStrip()` — pure rendering from `state`.
- `moveTask(taskId, newStatus)` — used by both the native HTML5 drag-and-drop handlers and the keyboard-accessible "Move ▸" `<select>` fallback on each card. Keep both entry points calling this one function. It no-ops if `newStatus` equals the task's current status (avoids no-op audit log entries).
- `addTask` flow — `handleFormSubmit()` validates via `validateTask()`, generates the next `UOB-ITPM-####` id via `nextTaskId()`/`idCounter`, pushes to `state.tasks`, logs the creation via `logAudit()`, and renders *before* awaiting `notifyNewTask()` (optimistic UI: the card must appear regardless of network outcome).
- `deleteTask(taskId)` — gated by `pendingDeleteId`, which drives the inline "Delete? Yes / No" confirmation row rendered inside the card (no native `confirm()`).
- `escapeHtml(str)` — every user-supplied string (title, description, assignee, etc.) must pass through this before being interpolated into template-string HTML. There is no other sanitization layer.
- Board-level click/change events are handled via delegation on `#board` using `data-action`/`data-task-id` attributes, not per-card listeners — new card-level controls should follow this pattern rather than attaching inline `onclick`.
- Drag-and-drop listeners (`dragover`/`dragleave`/`drop`) are bound once per column body (`#body-<Status>`) at init, since those elements are never replaced — only their `innerHTML` is re-rendered.

Columns are the fixed set `["Backlog", "In Progress", "Blocked", "Done"]` (the `COLUMNS` constant); column DOM ids are `body-<Status>` / `count-<Status>` with the status string used verbatim (including the space in `"In Progress"`).

Priority-to-color mapping uses two independent class namings that must stay in sync with the CSS: `priority-<level>` on `.card` (for the left border) and `pill-<level>` on the priority `<span class="pill">` (for pill background/text color) — see the `--critical`/`--high`/`--medium`/`--low` custom properties in `:root`.

### Management Dashboard

`#tab-board` / `#tab-dashboard` toggle the `hidden` attribute on `#view-board` / `#view-dashboard` via `switchView()` — a plain visibility swap, not a router. The dashboard reads `state.tasks` / `state.auditLog` directly (it ignores board filters by design: it answers "how is the whole portfolio doing").

`renderDashboard()` is the dashboard's equivalent of `renderBoard()` — the only place that rebuilds dashboard DOM — and is always called from inside `renderBoard()`, so every task mutation keeps both views current even while the dashboard tab is hidden. It calls four focused renderers, in a deliberate top-to-bottom order (most-urgent-first, for scanning in a standup):
- `renderKpiRow()` — reads `computeKpis()` and renders the stat tiles (`#kpi-row`).
- `renderPriorityQueue()` — the dashboard's headline panel (`#priority-queue-list`), sits directly under the KPIs. Every open (non-Done) task, sorted overdue-first, then by `PRIORITY_RANK` (Critical → Low), then by `dueDate` — this is the ordering that makes "highlight critical items on top" true without a separate section. Capped at `PRIORITY_QUEUE_LIMIT` (6) with a "+N more" footnote; rows where `isOverdue(t) || t.priority === "Critical"` get the `.pq-urgent` red highlight. Each row shows a stage badge (`stageClass(status)` → `.stage-Backlog` / `.stage-In-Progress` / `.stage-Blocked` / `.stage-Done`) so the current column is visible without switching tabs — keep this in sync with `COLUMNS` if statuses ever change.
- `renderPortfolioCharts()` — three horizontal bar charts (status / priority / project mix), laid out side by side via `.charts-row` (not stacked) to stay compact, built as plain HTML/CSS bars via `renderBarChart(containerId, rows)`, not SVG or a charting library. Bar color follows the same job the data is doing: status and project bars use a single brand hue (magnitude by category, identity carried by the label), priority bars reuse the existing `--critical`/`--high`/`--medium`/`--low` tokens (identity already established by the pills elsewhere in the app) — don't invent a new categorical palette for these.
- `renderAuditLog()` — renders `state.auditLog` (newest first) into `#audit-log-list`.

`logAudit(type, task, detail)` appends an entry to `state.auditLog` (type is `"created" | "moved" | "deleted"`); it's called from `handleFormSubmit()`, `moveTask()`, and `deleteTask()` — any new task-mutating action should call it too. The log is capped at `MAX_AUDIT_ENTRIES` (200) and, like the rest of `state`, is in-memory only and resets on refresh.

The dashboard is intentionally dense/compact (small type scale, tight spacing) so a full standup — KPIs, priority queue, charts, audit log — fits in roughly one screen on a shared monitor; don't loosen the spacing back up without re-checking that it still reads as one page.
