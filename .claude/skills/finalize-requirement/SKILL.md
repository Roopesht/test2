---
name: finalize-requirement
description: Writes req.md and mock.md for a Requirement that was gated by add-requirement, using the now-answered qna clarifications. Refuses to run while the Requirement is still gated.
tags: [requirements, ai-generation, qna]
---

# Finalize Requirement (after QnA answered)

Usage: `/finalize-requirement <requirement-id>`

## What this does
Companion to `add-requirement.md`. When that skill finds a genuine gap, it
creates a Requirement + a `qna` document but deliberately does **not** write
`req.md`/`mock.md` yet. This skill is the second half: once someone answers
the qna, it drafts and commits the documents using the clarified answers.

## Steps

0. **Resolve the API base URL.** Read `apiBaseUrl` from this workspace's
   `.project-knm/context/config.json` (written by the extension at
   activation - it always matches whichever backend this install of the
   extension talks to). Do not fall back to a default - if the file is
   missing or `apiBaseUrl` is empty, stop here and report that the
   extension needs to run at least once in this workspace first, rather
   than guessing a host. Use the value as `$API_BASE_URL` in every call
   below.

1. **Check the gate first.**
   `GET $API_BASE_URL/api/gates?type=requirement&id=$ARGUMENTS`
   If `gatedYn` is `true`, **stop here** — do not draft or write anything.
   Fetch the qna document (`GET $API_BASE_URL/api/qna?entityType=requirement&entityId=$ARGUMENTS`)
   and report back exactly which `required` questions still have
   `answer: null`, so the user knows what's blocking it.

2. **Read the requirement.**
   `GET $API_BASE_URL/api/requirements/$ARGUMENTS`
   Note its `title`, `description`, `decisionId`, `projectId`.

3. **Read the qna document's answers, if one exists.**
   `GET $API_BASE_URL/api/qna?entityType=requirement&entityId=$ARGUMENTS`
   (404 is fine — it means this requirement was never gated in the first
   place; proceed using only the context from step 4.) Use every answered
   question's `answer` as ground truth for whatever it clarified.

4. **Re-gather the original context**, same as `add-requirement.md` steps
   1-3: the decision (`GET .../api/decisions/{decisionId}`), its parent
   feature/change request, and the project's blueprint if attached. This is
   needed again because the qna answers only make sense alongside the
   original context that prompted them.

5. **Draft the documents.** Using the requirement + decision + parent
   feature + blueprint + qna answers together, write:
   - `req.md`: the full requirement description in markdown, with every
     previously-open question now resolved using its qna answer (don't
     leave placeholder gaps — that's the whole point of this step existing)
   - `mock.md`: an ASCII mock/wireframe in a markdown code fence

   If the qna's answers still leave something genuinely unresolved (e.g. a
   question was optional and left unanswered, but turns out to matter), do
   not guess — stop and report what's still missing instead of writing a
   document with fabricated content.

6. **Write and commit the documents**, per `docs/file_structure.md`'s
   convention:
   - `req.md` -> `docs/requirements/{id}.md`
   - `mock.md` -> `docs/requirements/{id}/mock.md`

   Commit both via `POST $API_BASE_URL/api/documents/upload`
   (multipart: `entityType=requirement`, `entityId=<id>`, `key=doc` for
   req.md / `key=mock-ascii` for mock.md, `path=<path above>`,
   `file=<content>`). Header: `user-id: <decision's createdBy>`.

7. **Report back**: the paths committed, and a short note confirming which
   previously-open questions were incorporated into the final `req.md`.

## Notes
- `$API_BASE_URL` is resolved in step 0 from this workspace's
  `.project-knm/context/config.json` (`apiBaseUrl` field) - no fallback and
  no hardcoded host, so calls always hit whatever backend this install of
  the extension is actually configured against.
- This skill is idempotent-ish but not version-aware: running it twice just
  re-commits the documents (overwriting the prior version's content on
  GitHub, which keeps full history there). Fine to re-run if the requirement
  or its answers change later.
