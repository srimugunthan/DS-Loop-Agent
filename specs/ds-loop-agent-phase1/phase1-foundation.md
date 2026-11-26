# Phase 1: Foundation

**Depends on:** None (can start immediately)
**Status:** Not started

## Design components touched

Backend (FastAPI) and project store (§2, §6; D18, D20), REST + SSE contracts (§4; D33), browser UI shell, file browser and Monaco viewer (§2; D34), `start.sh` (D17).

## Objective

A local app where the user creates a project, sees the stepper (the Project step plus five stages), chats with a **fake** agent, and watches files appear live in a read-only file browser. Double-clicking a file opens it in a viewer tab. Everything later phases build on is in place: the project folder with numbered versions and JSONL logs, REST for actions, SSE for events, and the right-pane browser.

## Independently testable when

- `start.sh` starts the backend and frontend; the app is reachable on `127.0.0.1` only.
- A fake agent writes files into a stage folder; they appear in the tree without a refresh; a folder link in its message expands that folder; double-clicking opens a correctly rendered tab.
- Path-traversal tests on the file endpoints pass; the platform-only folder is never listed.

## Tasks

### 1.1 Create and open a project

- **Type:** AFK
- **Blocked by:** None
- **PRD reference:** §5.3 (header, stepper)
- **System design reference:** §2 Backend, §6 Storage; D17, D18, D20

**Scope:** FastAPI skeleton and a React app. "New project" creates a project folder with JSONL logs. The UI shows the header (app name, project name, Saved) and the stepper: Project plus five stages, all *not started*. A `start.sh` starts both. Not included: Docker Compose (Phase 2), agents.

**Acceptance criteria:**
- [ ] Creating, listing and reopening a project works across a backend restart.
- [ ] The project folder and its JSONL log exist on disk; the server binds to `127.0.0.1` (D41).
- [ ] Demoable in isolation.

### 1.2 Fake agent chat round trip over REST + SSE

- **Type:** AFK
- **Blocked by:** 1.1
- **PRD reference:** §5.1, §5.3 (chat thread, message box)
- **System design reference:** §4 UI actions / UI events; D33

**Scope:** Sending a message is a REST call. A fake echo agent replies through an SSE stream; messages are grouped by day with sender and time. Enter sends; Shift+Enter adds a new line. Every message is appended to the JSONL log. Not included: a real agent.

**Acceptance criteria:**
- [ ] Messages stream into the chat pane through SSE, and the stream reconnects after a dropped connection.
- [ ] Each message is in the JSONL log (D14).
- [ ] Demoable in isolation.

### 1.3 Read-only file tree confined to the project folder

- **Type:** AFK
- **Blocked by:** 1.1
- **PRD reference:** §5.3 (file browser)
- **System design reference:** §4 UI actions (read-only list/read); D34

**Scope:** Read-only list and read endpoints, confined to the project folder, and a tree component in the right pane (the default view). The platform-only folder (later: held-out data, gate outputs) is never exposed. Not included: live refresh (1.4), file tabs (1.5).

**Acceptance criteria:**
- [ ] Tests: `..`, absolute paths and symlinks outside the project are refused; the platform folder is never listed or readable.
- [ ] There are no write endpoints for files.
- [ ] Demoable in isolation.

### 1.4 Live refresh and folder links

- **Type:** AFK
- **Blocked by:** 1.2, 1.3
- **PRD reference:** §5.3, §7 step 7
- **System design reference:** §4 UI events (file changed); D34

**Scope:** A file watcher on the project folder sends file-changed events over SSE, and the tree updates. The fake agent can reply "Output is in `stages/data-prep/v001/`", rendered as a link that switches to the file browser and expands that folder.

**Acceptance criteria:**
- [ ] A file written by the fake agent appears in the tree within about 1 second, with no manual refresh.
- [ ] Clicking a folder link expands and highlights that folder.
- [ ] Demoable in isolation.

### 1.5 File tabs: Monaco viewer, previews, diff view

- **Type:** AFK
- **Blocked by:** 1.3
- **PRD reference:** §5.3 (file tabs; the mockup's markup-leak defect); §5.1 Copy and Download
- **System design reference:** D34

**Scope:** Double-clicking opens a tab with a read-only Monaco editor (line numbers, highlighting). Previews by type: CSV shows the first 100 rows as a table, images display, Markdown is rendered, and large or binary files show a size notice. A `diff.patch` opens in Monaco's side-by-side diff view. Copy and Download give the raw file. There is a footer showing freshness and size.

**Acceptance criteria:**
- [ ] Source renders exactly as written: no highlighter markup leaks into the code (test with a file containing HTML-like strings).
- [ ] Each preview type and the diff view work on sample files.
- [ ] Demoable in isolation.

### 1.6 Version folders labelled approved / latest

- **Type:** AFK
- **Blocked by:** 1.3
- **PRD reference:** §5.3 (file browser)
- **System design reference:** §3 Version; D20, D34

**Scope:** The store creates numbered version directories per stage (`vNNN`), and the tree labels the *latest* and the *approved* version. Approval itself comes in Phase 4; here a test endpoint or fixture marks a version as approved.

**Acceptance criteria:**
- [ ] New versions get the next number; existing versions are never overwritten.
- [ ] Labels render correctly and update over SSE.
- [ ] Demoable in isolation.
