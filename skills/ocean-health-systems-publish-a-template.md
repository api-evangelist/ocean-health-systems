---
name: ocean-health-systems-publish-a-template
description: >-
  Import or update an openEHR template in CKM safely — validate first, use conditional requests to
  avoid clobbering someone else's revision, and understand which changes the API refuses. Write
  surface; requires credentials and project rights.
generated: '2026-09-02'
method: generated
source: openapi/ocean-health-systems-ckm-rest-api-openapi.json
api: CKM REST API 1.6.0
base_url: https://ckm.openehr.org/ckm/rest/v1
operations:
  - signIn
  - getCurrentUser
  - getValidationReportForTemplate
  - importTemplate
  - getTemplateAsCKMResource
  - getTemplateOETHash
  - updateTemplateOnTrunk
  - updateTemplateStatus
  - getCurrentTemplateStatus
  - listRequiredArchetypesForTemplate
  - signOut
---

# Publish a template to CKM without breaking anything

This flow writes to a governance repository other people review against. Treat every step as
consequential.

## 1. Authenticate

Either send HTTP Basic on each call, or exchange it once: `signIn` (`POST /sessions`) with the
Authorization header set, then carry the returned session id in the **`JSESSIONID` request header**
(it is declared as an `apiKey` in `header`, not as a cookie). `getCurrentUser` (`GET /sessions`)
confirms who you are. `signOut` (`DELETE /sessions`) when done. Sessions expire — on `401` with
"the supplied session is not valid or has already expired", re-authenticate rather than retrying.

## 2. Dry-run the OET — always

`getValidationReportForTemplate` (`POST /templates/validation-report`) takes the OET body and
returns CKM's validation report **without importing anything**. Ocean's own contract recommends
calling it before both import and update. It is a POST rather than a GET only because the OET is too
long for a URI. Pass `include-information=true` to get informational findings as well as errors.

Read `listRequiredArchetypesForTemplate` on the target if you are updating — a template that depends
on outdated resources will fail with `424 Failed dependency`.

## 3a. Import a new template

`importTemplate` (`POST /templates`) with the OET as the body and:
- `cid-project` — the project that will own it. You must hold upload rights there or you get `403`.
- `log-message` — **required**, and written into the revision history. Make it meaningful.
- `template-type` — `NORMAL`, `ORDER_ITEM`, `ORDER_SET` or `KNOWLEDGE_TOPIC`.
- `proceed-if-outdated-resources-used` — only set true when the outdated dependency is deliberate.

## 3b. Update an existing template

1. `getTemplateAsCKMResource` (`GET /templates/{cid-template}`) to read the current
   `versionAsset` and `modificationTime`.
2. `getTemplateOETHash` (`GET /templates/{cid-template}/hash`) for the MD5 validator.
3. `updateTemplateOnTrunk` (`PUT /templates/{cid-template}`) with the new OET, a `log-message`, and
   **`if-match` set to that hash** (or `if-unmodified-since` set to the modification time).
   `republish-immediately` controls whether the change goes live at once.

On `412 Precondition failed` somebody else changed the template while you worked. **Do not retry the
same request.** Re-read, re-apply your change to the new revision, and resubmit with a fresh
`if-match`.

## 4. Move the status

`updateTemplateStatus` (`PUT /templates/{cid-template}/status`) with the `status` query parameter.
It also accepts `if-match` / `if-unmodified-since`, and `proceed-if-active-branches` for the case
where branches are open. `getCurrentTemplateStatus` reads the current value;
`getTemplateStatusAtVersion` reads it at a given asset version.

## What this API will refuse

- Published archetypes cannot be updated through the API at all — the contract says only
  *unpublished* trunk archetypes can. Same for forks: copyright and references need manual
  adaptation, so use the CKM web UI.
- An update must not change the copyright or the original namespace, and must not be byte-identical
  to the current revision. Both return `400`.
- Branch resources have no status; asking for one returns `403`.

## Retry and reversal — read this before you automate it

There is **no idempotency key**. If `importTemplate` times out, you cannot tell CKM that your retry
is the same import — list and check before resubmitting.

There is **no undo**. `deleteTemplate` and `deleteArchetype` are documented by Ocean as
"PERMANENTLY and IRREVOCABLY" destroying the resource *and everything that depends on it* —
branches, review rounds, individual reviews, discussion comments, resource-centre documents, to-do
tasks and change requests. No restore operation exists and no retention window is published. Never
put a delete behind an unattended agent. See the `reversibility` block in
`conventions/ocean-health-systems-conventions.yml`.
