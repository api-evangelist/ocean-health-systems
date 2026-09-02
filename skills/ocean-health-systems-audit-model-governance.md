---
name: ocean-health-systems-audit-model-governance
description: >-
  Report on the governance state of a CKM instance — which clinical models are published versus
  draft, what change requests and resource proposals are open, and how a model's status has moved
  across its asset versions. Read-only.
generated: '2026-09-02'
method: generated
source: openapi/ocean-health-systems-ckm-rest-api-openapi.json
api: CKM REST API 1.6.0
base_url: https://ckm.openehr.org/ckm/rest/v1
operations:
  - listSubdomains
  - listProjectsOfSubdomain
  - listProjects
  - getProject
  - listArchetypes
  - listTemplates
  - getCKMResource
  - getCurrentStatus
  - getStatusAtVersion
  - getCurrentArchetypeStatus
  - getArchetypeStatusAtVersion
  - getCurrentTemplateStatus
  - getTemplateStatusAtVersion
  - listChangeRequests
  - getChangeRequest
  - listResourceProposals
  - getResourceProposal
  - getProjectOfArchetype
  - getProjectOfTemplate
---

# Audit governance state in a CKM instance

CKM's value is the review lifecycle around clinical models, and that lifecycle is fully readable.
This flow produces a governance snapshot without writing anything.

## 1. Map the instance

`listSubdomains` (`GET /subdomains`) → `listProjectsOfSubdomain`
(`GET /subdomains/{cid-subdomain}/projects`), or `listProjects` (`GET /projects`) for everything at
once. `Project` tells you `projectType` (`PROJECT` or `INCUBATOR`), whether it is `public`, and
whether it is a `remoteSubdomain`. Incubators are where unstable work lives — separate them in any
report.

## 2. Inventory the resources

`listArchetypes` and `listTemplates`, scoped with `cid-project` or `cid-subdomain`. Filter by
`status` and `resource-state`, and use `get-latest-published` when you want the governed view rather
than the working view. Page with `offset` / `size` (no total count is returned — page until short).

## 3. Read the maturity signal, not just the status string

For each resource, `CkmResource` carries three version tracks at once:

| field | meaning |
|---|---|
| `versionAsset` | the revision you fetched |
| `versionAssetLatest` | the newest revision, published or not |
| `versionAssetLatestPublished` | the newest **reviewed and published** revision |

with `revision` / `revisionLatest` / `revisionLatestPublished` as the SemVer equivalents. A large
gap between `versionAssetLatest` and `versionAssetLatestPublished` is drift: work happening that
consumers cannot yet rely on. That gap is the single most useful number this API gives an auditor.

`branchName` being set means the resource is not on the trunk.

## 4. Trace status over time

`getCurrentStatus` (`GET /resources/{cid-resource}/status`) and `getStatusAtVersion`
(`GET /resources/{cid-resource}/status/{asset-version}`) work for archetypes and templates alike;
the typed equivalents are `getCurrentArchetypeStatus` / `getArchetypeStatusAtVersion` and
`getCurrentTemplateStatus` / `getTemplateStatusAtVersion`. Walk asset versions backwards to build a
status timeline.

A `403` here usually means the resource is a **branch**, which has no status — treat it as a shape
condition, not a permissions failure.

## 5. Pull the open governance work

- `listChangeRequests` (`GET /change-requests`) with `cid-resource`, `cid-project`, `priority`,
  `status`, and `start-date` / `end-date`; `getChangeRequest` for detail.
- `listResourceProposals` (`GET /resource-proposals`) with `cid-project`; `getResourceProposal` for
  detail.

Ageing open change requests against published resources are the finding worth surfacing.

## 6. Attribute ownership

`getProjectOfArchetype` (`GET /archetypes/{cid-archetype}/project`) and `getProjectOfTemplate` close
the loop from a resource back to the team accountable for it.

## Caveats to state in any report you produce

- **`404` is ambiguous.** The contract says a resource may be absent *or* invisible to your
  credentials. An unauthenticated audit systematically undercounts private projects — say which
  identity you ran as.
- **No totals.** Counts are derived by exhausting pages, not read from the API.
- **Instance-scoped.** Every number belongs to one CKM deployment. Confirm which with
  `GET /resources/publisher-namespace`.
