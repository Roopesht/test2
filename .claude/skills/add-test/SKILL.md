---
name: add-test
description: Draft a Test Case from a Requirement, using the requirement's own docs and (when exactly one exists, or one is named explicitly) a specific Work Item's impl.md as context. If there's anything it can't determine confidently, creates a qna document instead of guessing and does NOT write test-plan.md yet - see finalize-test for the follow-up once answered.
tags: [tests, ai-generation, requirements, qna]
---

# Add Test (from Requirement)

Usage: `/add-test <requirement-id> [work-item-id]`

`work-item-id` is optional - see step 2 for when to pass it.

## What this does
Given a Requirement ID, drafts a Test Case from:
- The requirement's own docs (`req.md`, `mock.md` if present)
- A specific Work Item's `impl.md`, when relevant (see step 2)
- The project's blueprint document (if one is attached)

**Documents are gated on clarity, not just informationally flagged** - same
as `add-requirement.md`/`add-workitem.md`. If there's a genuine ambiguity,
this skill creates the Test Case row and a `qna` document (which gates it)
but deliberately does **not** write `test-plan.md` yet. Once the qna is
answered, run `/finalize-test <test-id>` to actually generate and commit
the test plan using the clarified answers.

If there's nothing ambiguous, this skill does everything in one pass: Test
Case + `test-plan.md`, no gate.

Like `add-workitem`, this skill produces **one** document (`test-plan.md` -
scenarios/steps/expected outcomes, not a UI mock), since a test case is a
verification task, not a user-facing spec.

Unlike `add-requirement`/`add-workitem`, there is **no status precondition**
to check here - confirmed by reading `docs/SHARED-WORKFLOW-IMPLEMENTATION.md`
before writing this skill: nothing requires the Requirement (or a linked
Work Item) to be in a particular status before a Test Case can be created.

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
   Note its `title`, `description`, `projectId`, `documentLinks`, and
   `createdBy`. (The first positional argument is always the requirement
   id - see usage above.)

2. **Decide whether to link a specific Work Item.**
   - If a second argument (`work-item-id`) was given, use it - fetch it
     (`GET $API_BASE_URL/api/work-items/{id}`) and confirm its
     `requirementId` matches `$1` (the requirement id from step 1). If it
     doesn't match, stop and report the mismatch rather than silently
     using it or silently ignoring it.
   - If no second argument was given, check how many Work Items exist for
     this requirement: `GET $API_BASE_URL/api/work-items?requirementId=$1`.
     If **exactly one** exists, use it automatically (it's the
     unambiguous implementation for this requirement). If **zero or
     several** exist, don't link any Work Item - proceed with just the
     Requirement's own context. Do not guess which of several Work Items
     is "the" one.

3. **Read the requirement's own docs, if any.** Its `documentLinks` (or
   `document_links`) may have a `doc` key (the `req.md`) and a `mock` key
   (the ASCII mock). Fetch each from GitHub (the project's `docsRepoUrl` +
   that path) if present. If absent, proceed with just the requirement's
   `title`/`description` - a valid, expected state, not an error.

4. **If step 2 resolved a Work Item, read its `impl.md`.** Its
   `documentLinks` should have a `doc` key pointing at the implementation
   plan (STORY-045). If the Work Item exists but has no `doc` key yet
   (i.e. it's still gated, unfinalized), **stop here** - report that the
   Work Item needs `/finalize-workitem <work-item-id>` run first, and do
   not proceed using an incomplete/nonexistent implementation plan as
   context. If no Work Item was resolved in step 2, skip this step
   entirely - that's a valid, expected state, not an error.

5. **Read the project's blueprint, if any.**
   `GET $API_BASE_URL/api/projects/{projectId}` - check its
   `documentLinks` for a `blueprint` key, fetch it from GitHub if present.
   If absent, proceed without it.

6. **Draft the test case in-memory** (do not write files yet). Using the
   requirement + its docs + (if resolved) the Work Item's `impl.md` +
   blueprint as context, work out:
   - A short, clear test case title
   - `test-plan.md` content: concrete test scenarios/steps and their
     expected outcomes, in markdown - derived from the requirement's
     acceptance criteria and (if present) the work item's implementation
     approach

   **Be honest about gaps.** If the requirement (and linked work item, if
   any) don't give enough to write concrete, verifiable test scenarios -
   e.g. the requirement is still vague, or genuinely contradictory - do not
   invent scenarios. Track it as an open question; it determines whether
   step 9 runs.

7. **Generate a unique Test Case ID.** Follow this project's slug
   convention: lowercase, hyphenated, prefixed `TST-`, derived from the
   title (e.g. "Verify Salary Range Normalization" ->
   `TST-verify-salary-range-normalization`). Confirm it doesn't already
   exist (`GET .../api/test-cases/{id}` should 404); if it collides, append
   `-2`, `-3`, etc.

8. **Create the Test Case.**
   `POST $API_BASE_URL/api/test-cases`
   Body: `{ "id": "<generated id>", "projectId": "<requirement's projectId>", "requirementId": "$1", "workItemId": "<resolved work item id from step 2, or omit entirely>", "title": "<title>", "description": "<short summary, or 'Pending clarification - see qna' if gapped>", "createdBy": "<requirement's createdBy>" }`

   `requirementId` is the only mandatory link - `workItemId` is optional
   and should be omitted from the body entirely when step 2 didn't resolve
   one (don't send `null` or a guessed value).

9. **If step 6 found NO open questions**, write and commit `test-plan.md`
   now, per `docs/file_structure.md`'s convention: `docs/test/{id}.md`.

   Commit via `POST $API_BASE_URL/api/documents/upload`
   (multipart: `entityType=testCase`, `entityId=<id>`, `key=doc`,
   `path=docs/test/{id}.md`, `file=<content>`). Header:
   `user-id: <requirement's createdBy>`.

   **If step 6 found open questions, skip this step entirely** - no
   documents get written in this run. Go to step 10.

10. **If step 6 surfaced any open questions**, create a qna document
    instead of writing documents:
    `POST $API_BASE_URL/api/qna/questions`
    Header: `user-id: <requirement's createdBy>`
    Body: `{ "entityType": "testCase", "entityId": "<test case id>", "assignedTo": "<requirement's createdBy>", "questions": [ { "id": "q1", "type": "text", "text": "<the actual gap you found>", "required": true }, ... ] }`

    This automatically gates the Test Case (STORY-040) - no separate call
    is needed to set that. (Unlike `add-workitem`, the Test Case row can
    always be created here even when gapped - `workItemId` being optional
    means there's no field this skill could fail to supply the way
    `add-workitem` sometimes can't supply `applicationId`.)

11. **Report back**: the new Test Case's id, whether a Work Item was linked
    (and which one, or why none was), and one of:
    - `test-plan.md` committed clean (no gaps), with the path, or
    - Gated pending clarification - list the questions asked, and tell the
      user to run `/finalize-test <id>` once they're answered.

## Notes
- `$API_BASE_URL` is resolved in step 0 from this workspace's
  `.project-knm/context/config.json` (`apiBaseUrl` field) - no fallback and
  no hardcoded host, so calls always hit whatever backend this install of
  the extension is actually configured against.
- Never fabricate a requirement, work item, or blueprint that doesn't
  exist - if something doesn't resolve, stop and report that rather than
  guessing at what it might have meant.
- This skill deliberately has no precondition-checking logic (see "What
  this does" above) - confirmed, not assumed, by reading
  `docs/SHARED-WORKFLOW-IMPLEMENTATION.md` before writing it.
- Document key used here (`doc`) is STORY-022's existing free text
  convention, same as `add-requirement.md`/`add-workitem.md`.
- See `finalize-test.md` for the follow-up step once a gated Test Case's
  qna is answered.
