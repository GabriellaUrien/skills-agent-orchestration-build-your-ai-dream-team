# Project Pulse dashboard — implementation plan

## Summary

Build Mona's Project Pulse dashboard as a small static app under `app/`, plus a VS Code launch configuration at `.vscode/launch.json` so learners can run it via the **Run Project Pulse Dashboard** command. Work is split between the Designer (visual layer) and the Coder (markup, data, launch config). Because file scopes do not overlap, most work runs in parallel; only the launch configuration and final verification are sequential. No build tooling. No agent stages, commits, or pushes.

## Contract (agreed up front to enable parallel work)

These decisions freeze the interface between Designer and Coder so both can work in parallel without touching each other's files.

- **Semantic structure** (rendered by Coder, styled by Designer):
  - Root wrapper: `<main class="dashboard">` containing a header and `<section class="dashboard__grid">`.
  - Each project renders as `<article class="project-card" data-priority="…" data-status="…">` with child slots for name (heading), owner (byline), status badge (`.project-card__status`), recent activity (paragraph), and priority indicator (`.project-card__priority`).
  - Required CSS hooks the Designer must target and the Coder must emit: `.dashboard`, `.project-card`. Additional BEM-style modifier hooks are permitted, but these two must be present verbatim.
- **Data contract** (`app/project-data.json`):
  - Top-level object with a `projects` array.
  - Each project has exactly these keys: `name`, `owner`, `status`, `recentActivity`, `priority`.
  - `status` values: `on-track`, `at-risk`, `blocked`, `shipped` (used both as text and as the `data-status` attribute).
  - `priority` values: `low`, `medium`, `high` (used both as text and as the `data-priority` attribute).
- **Runtime shape**: Coder loads `project-data.json` via `fetch('./project-data.json')` from `index.html`, then renders cards into `.dashboard__grid`.

## Ordered implementation steps

1. **Freeze the contract above.** (Planner deliverable — this document. No files touched.)
2. **Author sample data** — `app/project-data.json` (Coder).
3. **Author dashboard markup and rendering** — `app/index.html` (Coder).
4. **Author dashboard styles** — `app/styles.css` (Designer).
5. **Author VS Code launch configuration** — `.vscode/launch.json` (Coder).
6. **Verify integrated result** — Orchestrator runs the launch config, confirms `index.html` loads (not a directory listing) and that cards render with visible statuses and priorities.

## File assignments

| Step | File | Owner |
| --- | --- | --- |
| 2 | `app/project-data.json` | Coder |
| 3 | `app/index.html` | Coder |
| 4 | `app/styles.css` | Designer |
| 5 | `.vscode/launch.json` | Coder |

Each file has exactly one owner; scopes do not overlap.

## Designer responsibilities

- Owns `app/styles.css` only.
- Produce a polished dashboard look: card grid layout, readable typography, a clear spacing scale, and contrast that meets WCAG AA for body text and status badges.
- Style the required hooks `.dashboard` and `.project-card` explicitly.
- Include `border-radius` and `box-shadow` declarations on `.project-card`.
- Style status badges by `[data-status="…"]` and priority indicators by `[data-priority="…"]`; use color plus a shape, icon, or label so meaning is never color-only.
- Responsive layout: the card grid collapses to a single column below ~600px.
- Do not modify markup, data, or launch files. If a class hook is missing, ask the Orchestrator to route the change to the Coder.

## Coder responsibilities

- Owns `app/index.html`, `app/project-data.json`, and `.vscode/launch.json`.
- `app/project-data.json`:
  - Top-level `projects` array with 4–6 realistic sample projects.
  - Each entry has exactly `name`, `owner`, `status`, `recentActivity`, `priority`, matching the contract's allowed values.
  - Valid JSON, UTF-8, LF newlines, trailing newline.
- `app/index.html`:
  - Semantic HTML5 with `<main class="dashboard">` and `.project-card` elements as defined in the contract.
  - `<link rel="stylesheet" href="./styles.css">`.
  - A script that `fetch`es `./project-data.json`, iterates `data.projects`, and appends a `.project-card` per entry with `data-status` and `data-priority` attributes.
  - Render an empty-state message when the array is empty, and a visible error message if the fetch fails.
  - No bundler, framework, or npm dependency.
- `.vscode/launch.json`:
  - Strict JSON, **no comments**, no trailing commas.
  - `version` `"0.2.0"` and a single entry in `configurations`.
  - `name`: `"Run Project Pulse Dashboard"` (exact string).
  - `cwd`: `"${workspaceFolder}/app"` (exact string).
  - Launch method: run a static file server whose working directory is `app/` and open `index.html`, not a directory index. Recommended shape: `"type": "node"`, `"request": "launch"`, `"runtimeExecutable": "python3"`, `"runtimeArgs": ["-m", "http.server", "4173"]`, `"cwd": "${workspaceFolder}/app"`, `"console": "integratedTerminal"`, plus a `serverReadyAction` that opens `http://localhost:4173/index.html`.
  - The string `index.html` must appear in the file.

## Dependencies between steps

- Step 1 (contract) blocks Steps 2, 3, 4, and 5.
- Step 3 (`index.html`) depends on Step 2 (`project-data.json`) as its runtime `fetch` target, but only at execution time — the data shape is fixed by the contract, so authoring can start in parallel.
- Step 4 (`styles.css`) depends only on the CSS-hook contract, not on Step 3's file content.
- Step 5 (`launch.json`) depends on Step 3 existing at `app/index.html` for the URL target, but does not read that file.
- Step 6 (verification) depends on Steps 2–5 all being complete.

## Parallel work decisions

**Can run in parallel (Wave A):**

- Step 2 — Coder authoring `app/project-data.json`.
- Step 3 — Coder authoring `app/index.html`.
- Step 4 — Designer authoring `app/styles.css`.

These touch disjoint files and depend only on the contract, so the Orchestrator can dispatch them concurrently. Steps 2 and 3 are both Coder-owned but target separate files.

**Must run sequentially (Wave B):**

- Step 5 — Coder authoring `.vscode/launch.json`, after Step 3, because the launch config targets `app/index.html` and the Coder should confirm that path exists before finalizing.
- Step 6 — Orchestrator verification, last, because it exercises the integrated app end to end.

## Edge cases to handle

- Empty `projects` array → render a visible empty state ("No projects yet.") rather than a blank grid.
- `fetch` failure (for example, opening the page over `file://`) → surface a visible error inside `.dashboard` explaining that the app must be served over HTTP.
- Unknown `status` or `priority` value → fall back to a neutral badge style and warn in the console; do not throw.
- Long `name`, `owner`, or `recentActivity` strings → cards wrap gracefully and do not overflow the grid (Designer).
- Sub-600px viewport → grid collapses to a single column with preserved spacing (Designer).
- `.vscode/launch.json` common mistakes: JSON comments, trailing commas, missing `cwd`, or a URL that resolves to the directory listing instead of `index.html`.
- Port 4173 already in use → if the Coder picks a different port, the `serverReadyAction` URL must match.

## Validation expectations

- `python3 -m json.tool app/project-data.json` succeeds.
- `python3 -m json.tool .vscode/launch.json` succeeds.
- `app/project-data.json` has a top-level `projects` key and each item exposes `name`, `owner`, `status`, `recentActivity`, `priority`.
- `app/index.html` contains `project-card` and references `styles.css` and `project-data.json`.
- `app/styles.css` contains the selectors `.dashboard` and `.project-card`, plus `border-radius` and `box-shadow` declarations.
- `.vscode/launch.json` contains `"Run Project Pulse Dashboard"`, `"${workspaceFolder}/app"`, and `index.html`.
- Running the **Run Project Pulse Dashboard** launch configuration opens the dashboard UI, not a directory listing, and cards render with visible statuses and priorities.
- `bash scripts/validate-exercise.sh` still passes.

## Open questions

- **Port choice**: 4173 is proposed. Confirm it is free in the Codespace, or pick an alternate and update `serverReadyAction`.
- **Interactivity**: any sort or filter controls, or read-only cards for this iteration?
- **Sample data**: should the Coder invent representative project names and owners, or is there a canonical list?
- **Icons**: may the Designer inline SVGs for status and priority to keep the app fully static?
- **Dark mode**: is a `prefers-color-scheme: dark` variant in scope?
- **serverReadyAction**: open in VS Code's simple browser or externally?
