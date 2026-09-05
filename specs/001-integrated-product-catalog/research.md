# Research: Integrated Product Catalog

**Date**: 2026-09-05

All technical unknowns identified during planning are resolved below. Exact package patch versions
will be captured by the lockfile when implementation begins; compatible major lines are fixed here.

## Decision 1: Current CAP and Node.js Baseline

**Decision**: Use Node.js 24 LTS, CAP Node.js `@sap/cds` 10.x, ECMAScript modules, and compatible
`@cap-js/*` 3.x adapters.

**Rationale**: CAP 10 requires Node.js 22 or newer and recommends Node.js 24. CAP also warns against
mixing package generations, and new CAP projects are ESM-based. See [CAP June 2026 release
guidance](https://cap.cloud.sap/docs/releases/2026/jun26) and the [CAP release
schedule](https://cap.cloud.sap/docs/releases/).

**Alternatives considered**:

- Node.js 22: supported but already in maintenance LTS.
- Node.js 26: tested by CAP but not yet active LTS on the planning date.
- CAP 9: maintained generation rather than the current major for a new project.

## Decision 2: Persistence and Deployment Profiles

**Decision**: Use SQLite in memory for local development and most automated tests, SAP HANA Cloud
for production business data, and SAP BTP Cloud Foundry as the initial deployment target.

**Rationale**: CAP explicitly recommends SQLite for the local inner loop and SAP HANA for production;
the compatible database plugins are auto-wired by profile. See [using SQLite for development](https://cap.cloud.sap/docs/guides/databases/sqlite)
and [CAP database services](https://cap.cloud.sap/docs/guides/databases/new-dbs).

**Alternatives considered**:

- SQLite in production: rejected because it is not suitable for clustered cloud applications.
- PostgreSQL: viable in edge cases, but it adds a stack choice not required by this SAP-centered MVP.
- File-based development databases: useful for debugging but slower and less isolated than in-memory
  test databases.

## Decision 3: Live Association Instead of Product Replication

**Decision**: Persist one canonical `SupplierProduct`. `CatalogEntry` contains an association to the
product and the retailer-owned `isVisible` flag; service projections obtain current product fields
at read time. No propagation job, event, or snapshot is part of the MVP.

**Rationale**: This directly implements the constitution and makes supplier edits visible without a
second write path. Managed associations express references, while compositions are reserved for
objects whose lifecycle belongs to their parent. CAP recommends canonical UUID keys and concise
association-based models in its [domain modeling guide](https://cap.cloud.sap/docs/guides/domain/).

**Alternatives considered**:

- Copying supplier fields into each retailer entry: rejected because it creates conflicting sources
  of truth and synchronization work.
- Asynchronous replication events: rejected because there is no separate datastore or service in
  the MVP that requires replication.
- Runtime materialized snapshots: deferred until evidence shows a performance need.

## Decision 4: Domain Keys, Ownership, and Integrity

**Decision**: Apply CAP `cuid` and `managed` aspects to persistent entities. Use compositions only
for `SupplierProduct.images` and `RetailerCatalog.entries`; use associations for supplier, category,
retailer, and source product references. Enforce unique retailer catalog and catalog-product pairs
with `@assert.unique`, plus database referential integrity.

**Rationale**: UUIDs are CAP's recommended canonical keys. `managed` supplies lifecycle timestamps,
and unique constraints on managed associations compile to database constraints. See [common CAP
aspects](https://cap.cloud.sap/docs/cds/common), [domain modeling](https://cap.cloud.sap/docs/guides/domain/),
and [declarative constraints](https://cap.cloud.sap/docs/guides/services/constraints).

**Alternatives considered**:

- Composite primary keys: rejected because they complicate navigation and SAP Fiori elements edits.
- Composition from catalog entry to supplier product: rejected because deleting a catalog-owned
  object must never cascade to the supplier's canonical product.
- Custom UUID generation and timestamp handlers: rejected in favor of CAP generic behavior.

## Decision 5: Role-Specific OData V4 Services and Authorization

**Decision**: Expose `SupplierAdminService`, `RetailerAdminService`, and `PublicCatalogService` as
separate OData V4 services. Require `Supplier` or `Retailer` for administrative services, scope data
by a trusted organization identifier, and explicitly mark the public read-only service with the
CAP pseudo-role `any`. Use mocked identities only in development/test and JWT/XSUAA in production.

**Rationale**: CAP recommends declarative `@requires`/`@restrict` rules and use-case-specific
services. Production CAP protects endpoints by default, so anonymous public access must be explicit.
See [CAP authorization](https://cap.cloud.sap/docs/guides/security/authorization) and [CAP
authentication](https://cap.cloud.sap/docs/guides/security/authentication).

**Alternatives considered**:

- One broad service for all actors: rejected because it increases accidental data exposure and UI
  coupling.
- Authorization only in JavaScript handlers: rejected because declarative filters are easier to
  audit and harder to bypass.
- CAP multitenancy: deferred; suppliers and retailers are business records inside one MVP tenant.

## Decision 6: Platform-Managed Product Images

**Decision**: Use the official `@cap-js/attachments` plugin. Model each product image as one
attachment child with file metadata and presentation order. Use SAP BTP Object Store for production
content and the plugin-supported local backend for development. Validate a maximum of ten files,
10 MB each, and JPEG/PNG/WebP types in the service before accepting content.

**Rationale**: The plugin provides attachment modeling, streaming, malware-scan integration, and
managed storage backends without custom upload infrastructure. OData V4 media properties support a
metadata POST followed by a streamed content PUT. See the [CAP attachments
plugin](https://cap.cloud.sap/docs/plugins/#attachments) and [CAP media data
guide](https://cap.cloud.sap/docs/guides/services/media-data).

**Alternatives considered**:

- Direct `LargeBinary` persistence in business tables: simpler at first, but duplicates plugin
  behavior and SQLite cannot stream it.
- External supplier-provided URLs: rejected by the clarified specification.
- Custom object-store integration: rejected because it recreates upload, scan, and streaming logic.

## Decision 7: Optimistic Concurrency with ETags

**Decision**: Annotate `SupplierProduct.modifiedAt` with `@odata.etag`; clients must use the ETag in
`If-Match` for product updates. Return `412 Precondition Failed` for stale writes and require reload
and manual reapplication. Image mutation also advances the product aggregate version.

**Rationale**: CAP directly supports optimistic locking with ETags and identifies `managed.modifiedAt`
as a suitable version value. This matches the clarified behavior without long-lived locks. See [CAP
concurrency control](https://cap.cloud.sap/docs/guides/services/served-ootb#conflict-detection-using-etags).

**Alternatives considered**:

- Last-write-wins or `If-Match: *`: rejected because it silently loses confirmed changes.
- Pessimistic locks or exclusive drafts: rejected because they change the approved interaction and
  add lifecycle complexity.
- Automatic merge: rejected as out of scope for the MVP.

## Decision 8: Metadata-Driven SAPUI5 Administration

**Decision**: Build two SAP Fiori elements applications for OData V4. The supplier app uses List
Report/Object Page for product CRUD and image streams. The retailer app exposes supplier/product
browsing, an annotated `addToCatalog` action, and catalog-entry visibility management. Use custom
pages or controller extensions only for gaps that annotations cannot cover.

**Rationale**: Standard floorplans minimize application-specific JavaScript and keep enterprise UX
consistent. An SAPUI5 OData V4 model targets one service, supporting the role-specific service split.
See [SAP Fiori elements application guidance](https://help.sap.com/docs/SAP_FIORI_tools/17d50220bcd848aa854c9c182d65b699/7833775ae607430c9d708d9a3a145263.html),
[OData V4 model guidance](https://help.sap.com/docs/SAPUI5/3f47ec0c79a547aeaa23090b74c9520c/5de13cf4dd1f4a3480f7e2eaaee3f5b8.html),
and [Fiori elements actions](https://help.sap.com/docs/SAPUI5/b2f662dd9d7a4ec680056733050b4d34/cbf16c599f2d4b8796e3702f7d4aae6c.html).

**Alternatives considered**:

- Freestyle SAPUI5 applications: rejected because they require more code for standard CRUD flows.
- One administrative app with unrelated tabs: rejected because it mixes personas and data domains.
- CAP draft handling: rejected because the approved conflict behavior is non-draft optimistic
  concurrency.

## Decision 9: React Public Catalog and Same-Origin Delivery

**Decision**: Build a client-rendered React/TypeScript SPA using Vite, React Router, a small typed
`fetch` wrapper for the public OData subset, and TanStack Query for cache/freshness. Serve the built
app and OData route through the same SAP Application Router origin; use a Vite proxy locally.

**Rationale**: The public service is read-only and needs only selected OData query capabilities, so
a full browser OData SDK is unnecessary. Same-origin delivery removes production CORS complexity.
React supports Vite for custom client applications, while CAP advises centralizing CORS policy. See
[React application setup](https://react.dev/learn/build-a-react-app-from-scratch), [Vite getting
started](https://vite.dev/guide/), [CAP UI serving](https://cap.cloud.sap/docs/guides/uis/), and
[CAP CORS guidance](https://cap.cloud.sap/docs/node.js/best-practices#cross-origin-resource-sharing-cors).

**Alternatives considered**:

- A full OData browser SDK: rejected for the small read-only query surface.
- Fetch calls directly inside components: rejected because they duplicate loading, error, cache,
  and freshness behavior.
- SSR or a second frontend server: deferred until SEO or prerendering is proven necessary.
- Cross-origin deployment: rejected for the MVP; if later required, use one exact-origin allowlist.

## Decision 10: Freshness, Paging, and Caching

**Decision**: Filter public products on the server with `CatalogEntry.isVisible = true` and
`SupplierProduct.isActive = true`. Use default page size 20 and maximum 100. Revalidate public JSON
within 30 seconds and refetch an open catalog at least every 60 seconds and on window focus. Keep
media URLs relative and never embed binary content in list payloads.

**Rationale**: Server-side eligibility cannot be bypassed by the consumer. Bounded paging protects
the public service, and the freshness policy satisfies the 60-second success criterion without an
eventing system. CAP provides generic pagination and opaque `@odata.nextLink` handling; see [CAP
pagination](https://cap.cloud.sap/docs/guides/services/served-ootb#pagination-sorting).

**Alternatives considered**:

- Push updates or event streams: rejected because they add infrastructure without an MVP need.
- Unbounded list responses: rejected because they conflict with scale targets.
- Long immutable cache for catalog JSON: rejected because visibility and activation changes must
  become observable quickly.

## Decision 11: Layered Test Strategy

**Decision**: Use Vitest with `cds.test` and SQLite for backend and OData contract tests; QUnit for
authored SAPUI5 units; OPA5 for SAPUI5 journeys; Vitest, React Testing Library, and MSW for the public
app; and one thin real-browser end-to-end suite across supplier, retailer, and consumer flows. Add a
small hybrid HANA/Object Store suite for database constraints and streaming before release.

**Rationale**: CAP 10 is moving to Vitest/ESM, `cds.test` exercises actual HTTP services, and OPA5 is
the SAPUI5 integration-test facility. See [CAP testing](https://cap.cloud.sap/docs/node.js/cds-test),
[CAP 10 testing direction](https://cap.cloud.sap/docs/releases/2026/jun26#going-for-vitest-esm), and
[SAPUI5 testing guidance](https://help.sap.com/docs/SAPUI5/d625376e710e40cb9d40e43e1b02933b/7cdee404cac441888539ed7bfe076e57.html).

**Alternatives considered**:

- Jest for new CAP tests: rejected because CAP is moving away from it and ESM support is less direct.
- UI-only mock tests: rejected because authorization, ETags, projections, and media require the real
  CAP service contract.
- A large cross-browser E2E suite: deferred; one critical smoke flow gives better MVP feedback per
  maintenance cost.
