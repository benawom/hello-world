# Personal Life Management App — Design Spec

A single system for managing tasks, routines, projects, client work, content
production, relationships, and personal knowledge — built around one rule:
**capture must be frictionless, or the system dies.**

This spec describes the data model, screens, and integrations needed to build
the app. It does not prescribe a stack; pick whatever lets you ship a working
vertical slice fast (a Next.js/Node + Postgres app is a reasonable default).

## 1. Guiding principles

- **Low-friction capture first.** Any entry point (voice, text, keyboard
  shortcut) must get an item into the system in one step. Classification and
  cleanup happen after capture, not during.
- **One database, many views.** Tasks, routines, projects, content, people,
  and library items are distinct entities, not one undifferentiated pile.
  Mixing them back together is the failure mode this system exists to avoid.
- **Everything rolls up to a Domain.** Every project, area, task, and routine
  belongs to a top-level Domain (e.g. a business, a channel, "Home").
- **Nothing silently disappears.** Stale items resurface; ambiguous captures
  go to a review inbox instead of being dropped or mis-filed.

## 2. Core entities

### Domain
Top-level bucket everything else lives under (e.g. "Home", a company, a
YouTube channel, a client brand).
- `name`, `description`, `color/icon`

### Area vs. Project vs. Retainer
All three are "engagements" that own tasks, but behave differently.

| | Area | Project | Retainer |
|---|---|---|---|
| End date | none | yes | none (ongoing) |
| Recurs monthly | no | no | yes — tasks/checklist reset each cycle |
| Milestones | no | yes, with % complete | no |
| Typical use | ongoing non-client work | one-off deliverable with a deadline | ongoing client engagement |

Common fields: `name`, `domain_id`, `status`, `type` (area/project/retainer),
`checklist_template_id?`, `hours_logged`, `activity_log[]`.

Project-only: `end_date`, `milestones[]` (`name`, `percent_complete`).
Retainer-only: `billing_cycle_day`, `recurring_tasks[]`, `recurring_checklist_items[]`
(regenerate at cycle start; incomplete items from the prior cycle carry forward).

### Checklist Template
Reusable checklist (e.g. "New website build") that can be applied to any
project/retainer on creation instead of re-entering steps each time.
- `name`, `items[]` (`text`, `order`)

### Task
- `title`, `description?`, `status` (open/done), `priority`
- `due_date?`, `reminder(s)?` (push notification schedule)
- `project_id?` or `area_id?` (mutually exclusive link)
- `content_id?` (optional link to a Content item)
- `is_top3` (boolean flag for the day's Top 3)
- `recurrence?` (rule for auto-regenerating on completion)
- `source` (voice/text/manual/import) — for traceability back to capture

### Routine
Distinct from tasks. Two modes:
- **Daily recurring**: `time_of_day` (morning/afternoon/evening or specific
  time), `notify` (bool), tracked via a streak counter that resets on a
  missed day and shows "missed the window" if unchecked once its window has
  passed.
- **Fixed-length streak**: `start_date`, `duration_days`, same daily
  check-in mechanic; on completion (or abandonment) it moves to an
  **archive** with a bar chart of daily completion over the run.

Fields: `name`, `description?`, `time_of_day`, `notify`, `mode`
(recurring/fixed), `duration_days?`, `start_date?`, `streak_count`,
`check_ins[]` (date, completed bool), `archived_at?`.

### Content
For content production (video/article/podcast/newsletter).
- `title`, `type`, `channel`, `status` (idea/editing/waiting/published)
- `publish_date?`, `url?`, `embed_url?`
- `outline` (markdown)
- `checklist_id?` (outline → promotion steps)
- `linked_task_ids[]`

### Person (personal CRM)
- `name`, `relationship`, `birthday?`, `anniversary?`
- `notes[]` (freeform facts worth remembering — kids' names, interests)
- `interactions[]` (`date`, `note`) — a log, not a scored pipeline

### Library
- **Note**: `text`, `source?`, `tags[]`, `needs_review` (bool), `image?`
- **Quote**: `text`, `source_type` (book/article/podcast/conversation),
  `source_name`, `tags[]`, `thoughts[]` (append-only feed of reflections
  added over time — the quote accumulates meaning instead of staying static)
- **Journal Entry**: `text`, `date`, `media?` (photo/short clip), mostly
  populated via voice capture
- **Book**: `title`, `author`, `cover_url`, `status` (want-to-read/reading/
  finished/abandoned), `format`, `start_date?`, `finish_date?`, `rating?`,
  `notes?`; Kindle highlights import as individual Quotes linked to the book
- **Inventory item** (stretch): `name`, `photo`, `category`, `estimated_value?`,
  `in_use` (bool) — for insurance/stewardship tracking

### Resurfacing queue
A background job that selects one Library item (note/quote/journal entry)
per day to feature on the Today screen, plus flags Domains/Projects/Tasks
that haven't been touched in N days ("In Brief" resurfacing).

### Needs Review Inbox
Captures that the classifier couldn't confidently route land here with the
raw text, for manual filing.

## 3. Screens

### Today
1. **Top 3** — tasks flagged `is_top3`, checkable, promote any task via a
   star toggle.
2. **Calendar** — read-only feed pulled live from Google Calendar (this app
   does not own calendar data, only displays it).
3. **Open tasks** — everything else, sorted by due date.
4. **Resurfacing / In Brief** — stale-item nudges + the day's Library pick.
5. **Needs Review** — unfiled captures awaiting classification.

### Routines
Separate screen from Tasks. Grouped by time of day, streak indicator per
routine, visual flag when a window has passed unchecked. Archive view with
per-streak bar chart.

### Tasks
List/board sorted by open/done/all, grouped by project or area. Detail view
exposes project/area link, priority, content link, reminders, recurrence.

### Projects / Areas
List of engagements filtered by domain. Creation flow: choose Area vs.
Project vs. Retainer up front (behavior differs per §2). Project detail
shows milestones + checklist + hours log. Retainer detail shows current
cycle's recurring items + carry-forward from prior cycle.

### Content
Pipeline board by status. Detail view: embed, checklist, outline (markdown
editor), linked tasks, metadata (channel, publish date, URL).

### People
List + detail with notes and interaction log. No scoring/pipeline — a
memory aid, not a sales CRM.

### Library
Tabs: Notes, Quotes, Journal, Books, Inventory. Quote detail shows the
append-only thoughts feed.

### Domains
Top-level list; drilling into a domain shows its projects/areas/tasks/
routines.

### Settings
Integration health (Google Calendar, Pushover, Supabase/DB), manual
"force sync," global search, AI chat over the full dataset.

## 4. Capture flow

Every capture path funnels into one ingestion pipeline:

```
input (voice | text) -> transcribe (if voice) -> LLM classify+clean ->
  { entity_type, fields, confidence }
    -> if confidence high: create Task/Note/Journal/etc. directly
    -> else: create Needs Review Inbox item with raw text + best guess
-> push notification confirming what was created (or that review is needed)
```

Entry points:
- Watch shortcut → voice → transcription → same pipeline
- Phone: voice or text entry in-app
- Desktop: global keyboard shortcut (e.g. Cmd+J) opens a text capture box
- All entry points hit the same backend endpoint; the client is just a thin
  capture surface

## 5. Integrations

- **Google Calendar** — read-only sync via API for the Today screen; this
  app is not a calendar replacement.
- **Push notifications** — a service (e.g. Pushover) for task reminders,
  missed-routine alerts, and a daily summary; notifications also list inside
  the app.
- **LLM (Claude API)** — powers capture classification/cleanup and a
  chat-over-your-data feature (retrieval over Notes/Quotes/Journal/Tasks/
  People).
- **Auth** — the whole app sits behind username/password (or stronger)
  auth since it's reachable from the open internet.

## 6. Non-goals

- Not a calendar replacement — Google Calendar stays source of truth for
  events.
- Not a sales CRM — the People module is a memory aid, not a pipeline.
- Not multi-tenant — this is a single-user system per deployment.

## 7. Suggested build order (MVP → full)

1. Domains, Areas/Projects/Retainers, Tasks (CRUD + Today screen, no
   integrations)
2. Routines + streak tracking
3. Library (Notes/Quotes/Journal) + global search
4. Capture pipeline (text first, voice later) + Needs Review Inbox
5. Google Calendar read sync + push notifications
6. Content module + People module
7. Resurfacing engine + AI chat-over-data
8. Books/Kindle import + Inventory (stretch)
