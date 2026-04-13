# Taskt — CLAUDE.md

## Project Overview

**Taskt** is a single-file SPA task manager deployed on Vercel via GitHub auto-deploy.

- **Stack**: Vanilla HTML/CSS/JS — no build step, no framework, no bundler
- **File**: Everything lives in `index.html`
- **Auth + DB**: Supabase JS SDK (CDN), Google OAuth
- **Deploy**: Push to `main` → Vercel auto-deploys

---

## Architecture

### Single File Structure (`index.html`)
1. `<head>` — Google Fonts (Playfair Display, DM Sans), all CSS in `<style>`
2. `<body>` — Auth screen, app shell (sidebar + main content), all modals
3. `<script>` — Supabase config, then all app JS

### Key Constants (top of `<script>`)
```js
const SUPABASE_URL = 'https://npgsulhoxsvisleytivw.supabase.co';
const SUPABASE_ANON_KEY = '...';
const DEMO_MODE = SUPABASE_URL === 'YOUR_SUPABASE_URL'; // falls back to localStorage-only
```

---

## Data Model

### Tasks — stored in Supabase `public.tasks`
| Column | Type | Notes |
|---|---|---|
| `id` | uuid | primary key |
| `title` | text | |
| `notes` | text | |
| `priority` | text | `low / medium / high` |
| `due_date` | date | |
| `tags` | text[] | |
| `completed` | boolean | |
| `workspace` | text | workspace id |
| `page_id` | text | page id within workspace |
| `subtasks` | jsonb | array of `{id, title, done, mode}` |
| `subtask_mode` | text | default mode: `checkbox / bullet / number` |
| `images` | jsonb | array of base64 data URLs |
| `shared_with` | text[] | |
| `uid` | uuid | Supabase auth user id |

> **Required SQL migration** (run once in Supabase SQL Editor if columns missing):
> ```sql
> ALTER TABLE public.tasks
>   ADD COLUMN IF NOT EXISTS subtasks jsonb DEFAULT '[]',
>   ADD COLUMN IF NOT EXISTS subtask_mode text DEFAULT 'checkbox',
>   ADD COLUMN IF NOT EXISTS images jsonb DEFAULT '[]',
>   ADD COLUMN IF NOT EXISTS page_id text;
> ```

### Workspaces — stored in `localStorage` (`taskt_ws_<uid>`)
```js
[{
  id: 'ws-<timestamp>',
  name: 'string',
  icon: '💼',          // emoji
  color: '#e8d5a3',    // hex, used as --ws-color CSS var
  uid: '<supabase-uid>',
  pages: [{
    id: 'pg-<timestamp>',
    name: 'string',
    icon: '📄',
    color: '#e8d5a3'
  }]
}]
```

---

## Key Global State Variables
```js
let workspaces = [];
let currentWsId = 'all';    // 'all' = no workspace filter
let currentPageId = null;   // null = show all pages in workspace
let subtasks = [];           // subtasks for the open task modal
let subtaskMode = 'checkbox'; // current mode for NEW subtasks (per-item mode stored on each subtask)
let pastedImages = [];       // base64 images for open task modal
let currentFilter = 'all';  // all / today / upcoming / overdue
```

---

## UI Layout

### Left Sidebar
- Workspaces list (click to select, shows workspace color accent)
- Views: All Tasks, Today, Upcoming, Overdue
- Left/right slide toggle (no expand/collapse arrows — removed)
- No page management in sidebar

### Main Content Area
- **Workspace banner** — name, color, task count
- **Page tabs strip** (`#page-tabs-wrap`) — horizontal scrollable row with `‹ ›` arrows when overflow
  - Active tab: solid fill with page color + glow shadow
  - Inactive tabs: transparent bg, gray border, muted text
  - `＋ Page` button on the right
  - Default: first page auto-selected when switching workspace
  - New page auto-selected on creation
- **Filter tabs** — All / Upcoming / Today / Overdue
- **Quick-add input** — type + Enter or `+ Add` to create task
- **Task list** — cards with priority indicator, subtask progress bar

---

## Modals

### Workspace Modal (`#ws-modal-overlay`)
- Name, emoji picker (12 options), color picker (8 swatches)
- Create or edit workspace

### Page Modal (`#pg-modal-overlay`)
- Name, emoji picker (12 options), color picker (8 swatches)
- Triggered by `＋ Page` button in main content area
- `addPage(wsId)` / `closePgModal()` / `savePage()`

### Task Modal (`#task-modal`)
- Title, notes, due date, priority, tags
- Workspace + page dropdowns
- Reminders, share list
- Image paste (base64, stored in `images` jsonb)
- Subtasks section (see below)
- Discard confirmation if unsaved changes

---

## Subtasks

### Modes (per-item, stored on each subtask object)
| Mode | Appearance |
|---|---|
| `checkbox` | Square checkbox (3px border-radius), checkmark ✓ when done |
| `bullet` | • bullet prefix |
| `number` | 1. 2. 3. sequential prefix |

### Key rules
- The mode buttons (`☑ Checkbox`, `• Bullet`, `1. Number`) set `subtaskMode` for **newly added** items only
- Each subtask stores its own `mode` field — switching mode does NOT retroactively change existing subtasks
- Active mode button shows accent-colored border + text; inactive buttons are dimmed (opacity .4)
- `outline: none` on mode buttons prevents browser focus ring overriding the accent border

### Subtask data shape
```js
{ id: crypto.randomUUID(), title: 'string', done: false, mode: 'checkbox' }
```

---

## Important Functions

| Function | Purpose |
|---|---|
| `renderSidebar()` | Rebuilds sidebar + page tabs + modal dropdowns |
| `renderTasks()` | Filters + renders task cards |
| `selectWorkspace(id)` | Sets `currentWsId`, auto-selects first page |
| `selectPage(id)` | Sets `currentPageId`, re-renders |
| `addPage(wsId)` | Opens page creation modal |
| `savePage()` | Saves new page, auto-selects it |
| `deletePage(wsId, pageId)` | Removes page, clears selection if active |
| `persistTask(task)` | Upserts task to Supabase |
| `loadTasks()` | Fetches tasks from Supabase for current user |
| `updatePgScrollArrows()` | Shows/hides `‹ ›` scroll buttons based on overflow |
| `scrollPageTabs(dir)` | Scrolls page tabs strip left/right by 200px |
| `applySubtaskModeUI()` | Updates mode button styles (border + opacity) |
| `renderSubtasks()` | Renders subtask list using per-item `s.mode` |
| `addSubtask()` | Pushes new subtask with current `subtaskMode` stamped |

---

## CSS Conventions

- CSS custom properties on `:root` for all colors/spacing
- `--ws-color` set inline on `#main-content` for workspace theming
- Dark theme only (`--bg: #0f0f0f`)
- Fonts: Playfair Display (headings/modal titles), DM Sans (body)
- No external CSS framework

---

## Deployment

- **Repo**: `https://github.com/Surbhi23-ai/Taskt`
- **Branch**: `main` → auto-deploys to Vercel
- **Routing**: `vercel.json` rewrites all routes to `/index.html`
```json
{ "rewrites": [{ "source": "/(.*)", "destination": "/index.html" }] }
```

---

## Features Added (session log)

1. **Workspace colored sidebar** — each workspace has emoji + hex color, applied as CSS var
2. **Pages per workspace** — pages stored in workspace object in localStorage, each with name/icon/color
3. **Page creation modal** — custom styled modal (replaces browser `prompt()`), with emoji + color pickers
4. **Pages moved to main content** — page tabs render in main area, not sidebar
5. **Sidebar collapse toggles removed** — left/right slide toggle is sufficient; section collapse arrows removed
6. **Page tab active/inactive contrast** — active: solid color fill + glow; inactive: transparent + gray
7. **Auto-select first page** — switching workspace auto-selects first page; new page auto-selected on create
8. **Page tabs horizontal scroll** — overflow shows `‹ ›` arrow buttons instead of wrapping to new rows
9. **Subtask modes** — checkbox / bullet / number list, stored per-item so mixing modes is possible
10. **Subtask mode buttons as text** — replaced tiny icon buttons with labeled text buttons
11. **Square subtask checkbox** — `border-radius: 3px` (was `50%` circle/radio appearance)
12. **Image paste on tasks** — paste images into task modal, stored as base64 in `images` jsonb column
13. **Realtime sync** — Supabase realtime subscription for task updates
14. **Discard confirmation** — task modal warns before discarding unsaved changes
