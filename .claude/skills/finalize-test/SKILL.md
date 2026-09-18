---
name: finalize-test
description: Writes test-plan.md for a Test Case that was gated by add-test, using the now-answered qna clarifications. Refuses to run while the Test Case is still gated.
tags: [tests, ai-generation, qna]
---

# Finalize Test (after QnA answered)

Usage: `/finalize-test <test-id>`

## What this does
Companion to `add-test.md`. When that skill finds a genuine gap, it creates
the Test Case + a `qna` document but deliberately does **not** write
`test-plan.md` yet. This skill is the second half: once someone answers the
qna, it drafts and commits the test plan using the clarified answers.

(Simpler than `finalize-workitem.md` - `add-test` always creates the Test
Case row up front regardless of gaps, since `workItemId` being optional
means there's no required field it could ever be missing. There is no
"row doesn't exist yet" case to handle here.)

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
   `GET $API_BASE_URL/api/gates?type=testCase&id=$ARGUMENTS`
   If `gatedYn` is `true`, **stop here** - do not draft or write anything.
   Fetch the qna document (`GET $API_BASE_URL/api/qna?entityType=testCase&entityId=$ARGUMENTS`)
   and report back exactly which `required` questions still have
   `answer: null`, so the user knows what's blocking it.

2. **Read the test case.**
   `GET $API_BASE_URL/api/test-cases/$ARGUMENTS`
   Note its `title`, `description`, `requirementId`, `workItemId` (may be
   absent), and `projectId`.

3. **Read the qna document's answers, if one exists.**
   `GET $API_BASE_URL/api/qna?entityType=testCase&entityId=$ARGUMENTS`
   (404 is fine - it means this test case was never gated in the first
   place; proceed using only the context from step 4.) Use every answered
   question's `answer` as ground truth for whatever it clarified.

4. **Re-gather the original context**, same as `add-test.md` steps 1, 3-5:
   the requirement (`GET .../api/requirements/{requirementId}`), its own
   docs, the linked Work Item's `impl.md` if `workItemId` is set, and the
   project's blueprint if attached. This is needed again because the qna
   answers only make sense alongside the original context that prompted
   them.

5. **Draft `test-plan.md`.** Using the test case + requirement + (if
   linked) work item + blueprint + qna answers together, write concrete
   test scenarios/steps and expected outcomes, with every previously-open
   question now resolved using its qna answer (don't leave placeholder
   gaps - that's the whole point of this step existing).

   If the qna's answers still leave something genuinely unresolved (e.g. a
   question was optional and left unanswered, but turns out to matter), do
   not guess - stop and report what's still missing instead of writing a
   document with fabricated content.

6. **Write and commit `test-plan.md`**, per `docs/file_structure.md`'s
   convention: `docs/test/{id}.md`.

   Commit via `POST $API_BASE_URL/api/documents/upload` (multipart:
   `entityType=testCase`, `entityId=<id>`, `key=doc`,
   `path=docs/test/{id}.md`, `file=<content>`). Header:
   `user-id: <requirement's createdBy>`.

7. **Report back**: the path committed, and a short note confirming which
   previously-open questions were incorporated into the final test plan.

## Notes
- `$API_BASE_URL` is resolved in step 0 from this workspace's
  `.project-knm/context/config.json` (`apiBaseUrl` field) - no fallback and
  no hardcoded host, so calls always hit whatever backend this install of
  the extension is actually configured against.
- Idempotent-ish, not version-aware - same caveat as
  `finalize-requirement.md`/`finalize-workitem.md`: re-running just
  re-commits `test-plan.md` (overwriting the prior content on GitHub, which
  keeps full history there). Fine to re-run if the test case or its answers
  change later.
