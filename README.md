# UOB IT PMO — Kanban Board

A single-page, single-file Kanban board demo/training tool for an internal IT PMO. No build step, no framework, no backend beyond a FormSubmit notification hook — just open `index.html` in a browser.

## Run it

```bash
open index.html
```

## Notes

- All markup, styles, and logic live in `index.html`.
- Board state is in-memory only — refreshing the page resets it to seeded demo data.
- New task submissions optionally notify via [FormSubmit](https://formsubmit.co); update the `FORMSUBMIT_ENDPOINT` constant near the top of the `<script>` block to your own address.

See `CLAUDE.md` for architecture notes.
