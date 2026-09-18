---
name: finalize-workitem
description: Writes impl.md for a Work Item that was gated by add-workitem, using the now-answered qna clarifications. Also handles the case where add-workitem couldn't create the Work Item row at all yet (application was ambiguous) - creates it now that the qna is answered. Refuses to run while still gated.
tags: [work-items, ai-generation, qna]
---

# Finalize Work Item (after QnA answered)

Usage: `/finalize-workitem <id>`

`<id>` is normally a Work Item id. If `add-workitem` couldn't create the
Work Item row yet (application was ambiguous - see its step 9), pass the
**Requirement** id instead; this skill detects which case it is in step 1.

## What this does
Companion to `add-workitem.md`. When that skill finds a genuine gap, it
gates on a `qna` document instead of writing `impl.md` - and if the gap was
"which application does this belong to", it can't even create the Work Item
row yet (that field is required). This skill is the second half: once
someone answers the qna, it either (a) drafts and commits `impl.md` for an
existing gated Work Item, or (b) creates the Work Item now that the
application is known, then drafts and commits `impl.md`.

## Steps

0. **Resolve the API base URL.** Read `apiBaseUrl` from this workspace's
   `.project-knm/context/config.json` (written by the extension at
   activation - it always matches whichever backend this install of the
   extension talks to). Do not fall back to a default - if the file is
   missing or `apiBaseUrl` is empty, stop here and report that the
   extension needs to run at least once in this workspace first, rather
   than guessing a host. Use the value as `$API_BASE_URL` in every call
   below.

1. **Determine which case this is.**
   `GET $API_BASE_URL/api/work-items/$ARGUMENTS`
   - If this **200s**, `$ARGUMENTS` is a Work Item id → case (a), go to step 2.
   - If this **404s**, `$ARGUMENTS` is likely a Requirement id from the
     no-row-yet case → check
     `GET $API_BASE_URL/api/requirements/$ARGUMENTS` (should 200).
     If that also 404s, stop and report that the id doesn't resolve as
     either. Otherwise go to step 5 (case b).

## Case (a): Work Item already exists, gated

2. **Check the gate.**
   `GET $API_BASE_URL/api/gates?type=workItem&id=$ARGUMENTS`
   If `gatedYn` is `true`, fetch the qna
   (`GET $API_BASE_URL/api/qna?entityType=workItem&entityId=$ARGUMENTS`)
   and use its answers below. If `gatedYn` is `false`, there's nothing
   gating it - proceed anyway using whatever qna document exists (if none,
   just the Work Item + Requirement context).

3. **Re-gather context**, same as `add-workitem.md` steps 1, 3-6: the
   parent requirement (`requirementId` on the Work Item), its own docs, the
   decision/feature chain, the blueprint, and the applications list (to
   confirm the already-set `applicationId` still makes sense - it's fixed
   at this point, not re-askable).

4. **Draft and commit `impl.md`** using the requirement + docs + qna
   answers together, with every previously-open question now resolved (no
   placeholder gaps). Go to step 8.

## Case (b): no Work Item row exists yet, Requirement was gated instead

5. **Check the gate on the Requirement.**
   `GET $API_BASE_URL/api/gates?type=requirement&id=$ARGUMENTS`
   If still gated, fetch the qna
   (`GET $API_BASE_URL/api/qna?entityType=requirement&entityId=$ARGUMENTS`),
   find the application-selection answer, and confirm every `required`
   question is answered. If any required question is still `answer: null`,
   **stop here** and report exactly which ones.

6. **Create the Work Item now.**
   `GET $API_BASE_URL/api/projects/{projectId}/apps` to resolve the
   qna's answer to a real `applicationId` (match by name/id - don't
   fabricate one if the answer doesn't clearly match any listed app; stop
   and report instead).
   `POST $API_BASE_URL/api/work-items` with the same body shape as
   `add-workitem.md` step 9, now with a real `applicationId`.

7. **Re-gather context** (requirement's own docs, decision/feature chain,
   blueprint) and draft `impl.md`.

## Both cases

8. **Write and commit `impl.md`**, per `docs/file_structure.md`'s
   convention: `docs/workitem/{work-item-id}.md`.

   Commit via `POST $API_BASE_URL/api/documents/upload` (multipart:
   `entityType=workItem`, `entityId=<work item id>`, `key=doc`,
   `path=docs/workitem/{id}.md`, `file=<content>`). Header:
   `user-id: <requirement's createdBy>`.

9. **Report back**: the Work Item id (noting if it was just created in
   this run), the path committed, and a short note confirming which
   previously-open questions were incorporated.

## Notes
- `$API_BASE_URL` is resolved in step 0 from this workspace's
  `.project-knm/context/config.json` (`apiBaseUrl` field) - no fallback and
  no hardcoded host, so calls always hit whatever backend this install of
  the extension is actually configured against.
- Idempotent-ish, not version-aware - same caveat as
  `finalize-requirement.md`: re-running just re-commits `impl.md`
  (overwriting the prior content on GitHub, which keeps full history there).
