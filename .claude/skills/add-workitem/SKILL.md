---
name: add-workitem
description: Draft a Work Item from an approved Requirement, using the requirement's own docs, its parent decision/feature, and the project's blueprint. If there's anything it can't determine confidently (including which Application it belongs to), creates a qna document instead of guessing and does NOT write impl.md yet - see finalize-workitem for the follow-up once answered.
tags: [work-items, ai-generation, requirements, qna]
---

# Add Work Item (from Requirement)

Usage: `/add-workitem <requirement-id>`

## What this does
Given a Requirement ID, drafts a Work Item from:
- The requirement's own docs (`req.md`, `mock.md` if present)
- Its parent decision's `rationale` and parent feature/change request
  (title + description)
- The project's blueprint document (if one is attached)

**Documents are gated on clarity, not just informationally flagged** - same
as `add-requirement.md`. If there's a genuine ambiguity (including which
Application this work belongs to - a project can have several), this skill
creates the Work Item row and a `qna` document (which gates it) but
deliberately does **not** write `impl.md` yet. Once the qna is answered, run
`/finalize-workitem <work-item-id>` to actually generate and commit the
implementation plan using the clarified answers.

If there's nothing ambiguous, this skill does everything in one pass: Work
Item + `impl.md`, no gate.

Unlike `add-requirement`, this skill produces **one** document (`impl.md` -
an implementation plan, not a UI mock), since a Work Item is an engineering
task, not a user-facing spec.

## Steps

0. **Resolve the API base URL.** Read `apiBaseUrl` from this workspace's
   `.project-knm/context/config.json` (written by the extension at
   activation - it always matches whichever backend this install of the
   extension talks to). Do not fall back to a default - if the file is
   missing or `apiBaseUrl` is empty, stop here and report that the
   extension needs to run at least once in this workspace first, rather
   than guessing a host. Use the value as `$API_BASE_URL` in every call
   below.

1. **Read the requirement.** `GET $API_BASE_URL/api/requirements/$ARGUMENTS`
   Note its `title`, `description`, `decisionId`, `projectId`, `status`, and
   `documentLinks`.

2. **Check it's approved.** If `status` is not `approved`, **stop here** -
   do not create anything. Report back that the requirement must be
   `approved` first (current status included), and that a PM/lead needs to
   move it there before a Work Item can be created. Do not attempt to
   create the Work Item anyway - the API will reject it with a 409 if you
   try, so there's nothing to gain by trying.

3. **Read the requirement's own docs, if any.** Its `documentLinks` (or
   `document_links`) may have a `doc` key (the `req.md`) and a `mock` key
   (the ASCII mock). If present, fetch each file's raw content from GitHub
   (the project's `docsRepoUrl` + that path). If absent, proceed with just
   the requirement's `title`/`description` - a valid, expected state, not
   an error.

4. **Read the parent decision and feature/change request**, for extra
   context (same as `add-requirement.md` steps 1-2):
   `GET $API_BASE_URL/api/decisions/{decisionId}`, then
   `GET $API_BASE_URL/api/change-requests/{requestId}`.

5. **Read the project's blueprint, if any.**
   `GET $API_BASE_URL/api/projects/{projectId}` — check its
   `documentLinks` for a `blueprint` key, fetch it from GitHub if present.
   If absent, proceed without it.

6. **List the project's applications.**
   `GET $API_BASE_URL/api/projects/{projectId}/apps`
   Note each app's `id`, `name`, `description`.

7. **Draft the work item in-memory** (do not write files yet). Using the
   requirement + its docs + parent decision/feature + blueprint as context,
   work out:
   - A short, clear work item title
   - `impl.md` content: the implementation plan in markdown - approach,
     files/areas likely affected, and any open technical risks or
     assumptions worth flagging
   - Which application (from step 6's list) this work item belongs to

   **Be honest about gaps.** Two kinds matter here:
   - A genuine ambiguity in *what to build* (mirrors `add-requirement`'s
     gap handling)
   - **Which application it belongs to**, if it can't be confidently
     inferred from the requirement/decision/blueprint context (e.g. the
     requirement clearly touches both a backend API and a frontend, or the
     project has multiple apps and nothing points to one over another)

   Either kind is a real open question - do not guess. Track it; it
   determines whether step 10 runs.

8. **Generate a unique Work Item ID.** Follow this project's slug
   convention: lowercase, hyphenated, prefixed `WI-`, derived from the
   title (e.g. "Job Search Filters Endpoint" -> `WI-job-search-filters-endpoint`).
   Confirm it doesn't already exist (`GET .../api/work-items/{id}` should
   404); if it collides, append `-2`, `-3`, etc.

9. **Create the Work Item.**
   `POST $API_BASE_URL/api/work-items`
   Body: `{ "id": "<generated id>", "requirementId": "$ARGUMENTS", "applicationId": "<chosen app id, or omit if gapped - see note>", "projectId": "<requirement's projectId>", "title": "<title>", "description": "<short summary, or 'Pending clarification - see qna' if gapped>", "createdBy": "<requirement's createdBy>" }`

   `applicationId` is a required field on this endpoint - if step 7 found
   the application itself ambiguous, you cannot create the Work Item row
   yet without guessing. In that case, skip creating the row here; instead
   go straight to step 11's qna document, scoped to the Requirement
   (`entityType: "requirement"`, `entityId: "$ARGUMENTS"`) rather than the
   Work Item, since no Work Item exists yet to gate. Report this
   distinction clearly when you get to step 12.

   If the endpoint returns 409 ("Requirement must be approved first"),
   that means step 2's check was stale (status changed between your read
   and this call) - stop and report it, do not retry.

10. **If step 7 found NO open questions** (including application
    selection), write and commit `impl.md` now, per `docs/file_structure.md`'s
    convention: `docs/workitem/{id}.md`.

    Commit via `POST $API_BASE_URL/api/documents/upload`
    (multipart: `entityType=workItem`, `entityId=<id>`, `key=doc`,
    `path=docs/workitem/{id}.md`, `file=<content>`). Header:
    `user-id: <requirement's createdBy>`.

    **If step 7 found open questions, skip this step entirely** - no
    documents get written in this run. Go to step 11.

11. **If step 7 surfaced any open questions**, create a qna document
    instead of writing documents:
    `POST $API_BASE_URL/api/qna/questions`
    Header: `user-id: <requirement's createdBy>`
    Body: `{ "entityType": "workItem", "entityId": "<work item id>", "assignedTo": "<requirement's createdBy>", "questions": [ { "id": "q1", "type": "text", "text": "<the actual gap you found>", "required": true }, ... ] }`
    (Use `"entityType": "requirement"` and `"entityId": "$ARGUMENTS"`
    instead if step 9's application ambiguity meant no Work Item row exists
    yet - see the note there.)

    This automatically gates the entity (STORY-040) - no separate call is
    needed to set that.

12. **Report back**: the new Work Item's id (if one was created), and one of:
    - Documents committed clean (no gaps), with the path, or
    - Gated pending clarification - list the questions asked, which entity
      they're gating (Work Item, or the Requirement if no Work Item exists
      yet), and tell the user to run `/finalize-workitem <id>` once
      answered (only applicable once a Work Item id exists).

## Notes
- `$API_BASE_URL` is resolved in step 0 from this workspace's
  `.project-knm/context/config.json` (`apiBaseUrl` field) - no fallback and
  no hardcoded host, so calls always hit whatever backend this install of
  the extension is actually configured against.
- Never fabricate a requirement, decision, feature, blueprint, or
  application that doesn't exist - if something doesn't resolve, stop and
  report that rather than guessing at what it might have meant.
- This skill deliberately has **no other precondition-checking logic**
  beyond the one explicit approved-status check in step 2 - it doesn't
  second-guess the requirement's other fields or the decision's status. See
  STORY-038's "Design note for replicating this to WorkItem/Test/Decision"
  for why: STORY-042 (Workflow Rules Engine) is where that belongs
  long-term, enforced once at the shared API layer.
- Document key used here (`doc`) is STORY-022's existing free text
  convention, same as `add-requirement.md`.
- See `finalize-workitem.md` for the follow-up step once a gated Work
  Item's qna is answered.
