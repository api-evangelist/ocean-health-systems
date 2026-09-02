---
name: ocean-health-systems-discover-clinical-models
description: >-
  Find openEHR archetypes and templates in a Clinical Knowledge Manager instance and pull their
  machine-readable definitions (ADL, XML, OET, OPT) — the artefacts you feed to a CDR or a form
  generator. Read-only.
generated: '2026-09-02'
method: generated
source: openapi/ocean-health-systems-ckm-rest-api-openapi.json
api: CKM REST API 1.6.0
base_url: https://ckm.openehr.org/ckm/rest/v1
operations:
  - listArchetypes
  - getArchetypeAsCKMResource
  - getArchetypeInADL
  - getArchetypeInXML
  - getCiteableIdentifierForArchetypeId
  - getParentArchetype
  - listTemplates
  - getTemplateAsCKMResource
  - getTemplateOET
  - getTemplateOPT
  - listRequiredArchetypesForTemplate
  - listEmbeddedTemplates
  - getTemplateFileSetURL
  - listProjects
  - listSubdomains
---

# Discover clinical models in a CKM instance

CKM is Ocean Health Systems' governance repository for openEHR clinical content. It holds the
**models**, never patient data. Everything below is read-only.

## Before you start

- Pick the instance. The contract declares `basePath: /ckm/rest/v1` and **no host** — CKM is
  deployed per organisation. The public reference instance is `https://ckm.openehr.org/ckm/rest/v1`.
- Public projects on a public instance read without credentials. If you get `401`, authenticate
  (see `authentication/ocean-health-systems-authentication.yml`); if you get `404` on something you
  expect to exist, that may be a visibility failure rather than absence — the spec says so
  explicitly.
- Resources are addressed by **citeable identifier** (`cid`), e.g. `1013.1.130` — not by the openEHR
  archetype id.

## Steps

1. **Orient.** `listSubdomains` (`GET /subdomains`), then `listProjects` (`GET /projects`) to see
   what the instance governs. `GET /resources/publisher-namespace` (`PublisherNamespace`) confirms
   whose deployment you are on.

2. **Search.** `listArchetypes` (`GET /archetypes`) and `listTemplates` (`GET /templates`). Useful
   declared parameters: `search-text` with `require-all-search-words` and
   `restrict-search-to-main-data`; `class` with `include-subclasses` / `require-all-classes`;
   `resource-state` and `status`; `cid-project` / `cid-subdomain`; `language` / `locale`; and
   `offset` / `size` for paging.

3. **Page carefully.** The contract declares `offset` and `size` but **no total count and no next
   link**. Page until a response comes back shorter than `size`; do not assume a total.

4. **Resolve an id you already have.** If you hold an openEHR archetype id rather than a cid, call
   `getCiteableIdentifierForArchetypeId`
   (`GET /archetypes/citeable-identifier/{archetype-id}`); the template equivalent is
   `getCiteableIdentifierForTemplateId`.

5. **Fetch the definition.**
   - Archetype: `getArchetypeInADL` (`GET /archetypes/{cid-archetype}/adl`) or `getArchetypeInXML`.
   - Template: `getTemplateOET` for the authoring form, `getTemplateOPT`
     (`GET /templates/{cid-template}/opt`) for the **operational template** — that is what a CDR
     consumes. `getTemplateFileSetURL` returns a URL for the whole file set.

6. **Walk dependencies before you rely on a template.**
   `listRequiredArchetypesForTemplate` (`GET /templates/{cid-template}/required-archetypes`) and
   `listEmbeddedTemplates`. For specialisation lineage on an archetype, `getParentArchetype`.

7. **Pin a version.** Read operations accept `asset-version`, and `CkmResource` reports
   `versionAsset`, `versionAssetLatest` and `versionAssetLatestPublished` plus their SemVer
   `revision` counterparts. If you are generating code or forms, pin to
   `versionAssetLatestPublished` — the latest asset version may be an unreviewed draft.

8. **Cache with the hash.** `getArchetypeADLHash` (`GET /archetypes/{cid-archetype}/hash`) and
   `getTemplateOETHash` return MD5 validators. Use them to avoid refetching unchanged content, and
   keep them — they are the `if-match` values a write would need.

## Errors you will actually see

`400` malformed cid · `401` not authenticated or session expired · `403` often a *state* problem
(a branch resource has no status), not only a permissions problem · `404` absent **or** invisible.
Full catalogue: `errors/ocean-health-systems-problem-types.yml`. There is no error schema — you get
a status code and prose.

## Rate limits

None are published and none are declared in the contract; there is no `429` and no `RateLimit`
header. Self-throttle.
