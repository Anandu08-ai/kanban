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
const state = { tasks: [...], filters: { project, assignee, priority } };
```

`renderBoard()` is the only place that rebuilds card DOM — it re-derives the four columns from `state.tasks` (via `applyFilters()`) and calls `renderSummaryStrip()`. No code should mutate card contents directly; instead mutate `state` and call `renderBoard()`.

Key functions (each does one job):
- `renderBoard()` / `renderCard(task)` / `renderSummaryStrip()` — pure rendering from `state`.
- `moveTask(taskId, newStatus)` — used by both the native HTML5 drag-and-drop handlers and the keyboard-accessible "Move ▸" `<select>` fallback on each card. Keep both entry points calling this one function.
- `addTask` flow — `handleFormSubmit()` validates via `validateTask()`, generates the next `UOB-ITPM-####` id via `nextTaskId()`/`idCounter`, pushes to `state.tasks`, and renders *before* awaiting `notifyNewTask()` (optimistic UI: the card must appear regardless of network outcome).
- `deleteTask(taskId)` — gated by `pendingDeleteId`, which drives the inline "Delete? Yes / No" confirmation row rendered inside the card (no native `confirm()`).
- `escapeHtml(str)` — every user-supplied string (title, description, assignee, etc.) must pass through this before being interpolated into template-string HTML. There is no other sanitization layer.
- Board-level click/change events are handled via delegation on `#board` using `data-action`/`data-task-id` attributes, not per-card listeners — new card-level controls should follow this pattern rather than attaching inline `onclick`.
- Drag-and-drop listeners (`dragover`/`dragleave`/`drop`) are bound once per column body (`#body-<Status>`) at init, since those elements are never replaced — only their `innerHTML` is re-rendered.

Columns are the fixed set `["Backlog", "In Progress", "Blocked", "Done"]` (the `COLUMNS` constant); column DOM ids are `body-<Status>` / `count-<Status>` with the status string used verbatim (including the space in `"In Progress"`).

Priority-to-color mapping uses two independent class namings that must stay in sync with the CSS: `priority-<level>` on `.card` (for the left border) and `pill-<level>` on the priority `<span class="pill">` (for pill background/text color) — see the `--critical`/`--high`/`--medium`/`--low` custom properties in `:root`.
