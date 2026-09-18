---
name: add-requirement
description: Draft a Requirement from a Decision, using the decision text, parent feature, and the project's blueprint. If there's anything it can't determine confidently, creates a qna document instead of guessing and does NOT write req.md/mock.md yet - see finalize-requirement for the follow-up once answered.
tags: [requirements, ai-generation, decisions, qna]
---

# Add Requirement (from Decision)

Usage: `/add-requirement <decision-id>`

## What this does

Given a Decision ID, drafts a Requirement from:

- The decision's own text (`rationale`)
- Its parent feature/change request (title + description)
- The project's blueprint document (if one is attached)

**Documents are gated on clarity, not just informationally flagged.** If
there's a genuine ambiguity, this skill creates the Requirement row and a
`qna` document (which gates it) but deliberately does **not** write
`req.md`/`mock.md` yet — writing a document from an admittedly-incomplete
draft defeats the point of gating. Once the qna is answered, run
`/finalize-requirement <requirement-id>` to actually generate and commit the
documents using the clarified answers.

If there's nothing ambiguous, this skill does everything in one pass:
Requirement + both documents, no gate.

## Steps

0. **Resolve the API base URL.** Read `apiBaseUrl` from this workspace's
   `.project-knm/context/config.json` (written by the extension at
   activation - it always matches whichever backend this install of the
   extension talks to). Do not fall back to a default - if the file is
   missing or `apiBaseUrl` is empty, stop here and report that the
   extension needs to run at least once in this workspace first, rather
   than guessing a host. Use the value as `$API_BASE_URL` in every call
   below.

1. **Read the decision.** `GET $API_BASE_URL/api/decisions/$ARGUMENTS`
   Note its `requestId` (parent feature/CR), `rationale`, and `createdBy`.

2. **Read the parent feature/change request.**
   `GET $API_BASE_URL/api/change-requests/{requestId}`
   Note its `title`, `description`, and `projectId`.

3. **Read the project's blueprint, if any.**
   `GET $API_BASE_URL/api/projects/{projectId}` — check its
   `documentLinks` (or `document_links`) for a `blueprint` key. If present,
   fetch that file's raw content from GitHub (the project's `docsRepoUrl` +
   that path) for additional context. If absent, proceed without it — that
   is a valid, expected state, not an error to report.

4. **Draft the requirement in-memory** (do not write files yet). Using the
   decision + parent feature + blueprint (if available) as context, work out:
   - A short, clear requirement title
   - `req.md` content: the full requirement description in markdown
   - `mock.md` content: an ASCII mock/wireframe in a markdown code fence,
     illustrating the requirement's UI or flow if one is implied

   **Be honest about gaps.** If there is a genuine ambiguity you cannot
   confidently resolve from the available context (e.g. "should this field
   be required?", "which existing entity does this attach to?"), do not
   guess or invent an answer, and do not paper over it in the draft. Track
   it as an open question — it determines whether step 7 runs.

5. **Generate a unique Requirement ID.** Follow this project's slug
   convention: lowercase, hyphenated, prefixed `REQ-`, derived from the
   title (e.g. "Cart Checkout Validation" -> `REQ-cart-checkout-validation`).
   Confirm it doesn't already exist (`GET .../api/requirements/{id}` should
   404); if it collides, append `-2`, `-3`, etc.

6. **Create the Requirement.** Prefer the project-scoped endpoint over the
   generic `POST /api/requirements` — both set `requirements.decisionId`
   directly now (STORY-044 removed `requirement_versions`, so the tree views
   key off that column either way), but only the project-scoped endpoint
   derives `created_by` from the project's own creator; use it for
   consistency with the rest of the tooling.
   `POST $API_BASE_URL/api/projects/{projectId}/requirements`
   Body: `{ "id": "<generated id>", "title": "<title>", "description": "<short summary, or 'Pending clarification - see qna' if gapped>", "decision_id": "<decision id>" }`
   Note: this endpoint derives `created_by` from the project's own creator,
   not from the Decision's `createdBy` - you cannot set it explicitly here.

7. **If step 4 found NO open questions**, write and commit the documents now,
   per `docs/file_structure.md`'s convention (one main doc outside the
   entity's own folder, everything else inside it):
   - `req.md` -> `docs/requirements/{id}.md`
   - `mock.md` -> `docs/requirements/{id}/mock.md`

   Commit both via `POST $API_BASE_URL/api/documents/upload`
   (multipart: `entityType=requirement`, `entityId=<id>`, `key=doc` for
   req.md / `key=mock-ascii` for mock.md, `path=<path above>`,
   `file=<content>`). Header: `user-id: <decision's createdBy>`.

   **If step 4 found open questions, skip this step entirely** — no
   documents get written in this run. Go to step 8.

8. **If step 4 surfaced any open questions**, create a qna document for the
   new Requirement instead of writing documents:
   `POST $API_BASE_URL/api/qna/questions`
   Header: `user-id: <decision's createdBy>`
   Body: `{ "entityType": "requirement", "entityId": "<id>", "assignedTo": "<decision's createdBy>", "questions": [ { "id": "q1", "type": "text", "text": "<the actual gap you found>", "required": true }, ... ] }`

   This automatically gates the Requirement (STORY-040) — no separate call
   is needed to set that.

9. **Report back**: the new Requirement's id, and one of:
   - Documents committed clean (no gaps), with the paths, or
   - Gated pending clarification — list the questions asked, and tell the
     user to run `/finalize-requirement <id>` once they're answered.

## Notes

- `$API_BASE_URL` is resolved in step 0 from this workspace's
  `.project-knm/context/config.json` (`apiBaseUrl` field) - no fallback and
  no hardcoded host, so calls always hit whatever backend this install of
  the extension is actually configured against.
- Never fabricate a blueprint, decision, or feature that doesn't exist — if
  the given decision id doesn't resolve, stop and report that rather than
  guessing at what it might have meant.
- Document keys used here (`doc`, `mock-ascii`) are STORY-022's existing free
  text convention. STORY-023's stricter per-entity-type registry isn't built
  yet — once it lands, these keys must be validated against it.
- See `finalize-requirement.md` for the follow-up step once a gated
  Requirement's qna is answered.
