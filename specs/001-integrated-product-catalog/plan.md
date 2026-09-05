# Implementation Plan: Integrated Product Catalog

**Branch**: `001-integrated-product-catalog` (feature identifier; Git remains on `main`) |
**Date**: 2026-09-05 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `specs/001-integrated-product-catalog/spec.md`

## Summary

Build the Moveleira MVP as one SAP CAP Node.js application with a canonical supplier product model,
role-specific OData V4 services, two SAP Fiori elements administrative applications, and a React
public catalog. A retailer catalog entry stores only its relationship to the supplier product and
the retailer-owned visibility flag. Reads traverse that relationship, so supplier edits are visible
without copying or asynchronous synchronization. Product media uses the official CAP attachments
integration, and stale administrative edits are rejected through OData ETags.

## Technical Context

**Language/Version**: Node.js 24 LTS; ECMAScript modules; CDS with CAP 10.x; TypeScript 5.x for the
React application; JavaScript/XML for SAPUI5 extensions

**Primary Dependencies**: `@sap/cds` 10.x, compatible `@cap-js/*` 3.x adapters,
`@cap-js/attachments`, `@sap/xssec`, SAPUI5 1.151.x, SAP Fiori elements for OData V4, React 19.x,
React Router 7.x, TanStack Query 5.x, and Vite on its current Node.js 24-compatible stable line

**Storage**: SQLite in-memory for local development and automated backend tests; SAP HANA Cloud for
production business data; SAP BTP Object Store through `@cap-js/attachments` for production image
content; plugin-managed local attachment storage for development

**Testing**: Vitest with `cds.test` for CAP unit/integration/contract tests; QUnit and OPA5 for
SAPUI5-authored behavior and journeys; Vitest, React Testing Library, and MSW for React; a thin wdi5
or Playwright browser suite for the end-to-end MVP flow

**Target Platform**: Modern SAPUI5-supported desktop browsers for administration; responsive modern
browsers for the public catalog; Node.js service on SAP BTP Cloud Foundry behind SAP Application
Router; Linux-based local and CI environments

**Project Type**: Web platform monorepo with one CAP backend, two SAPUI5 administrative apps, and
one React public app

**Performance Goals**: 95% of public catalog list/detail views complete within 2 seconds; 95% of
supplier updates become observable in eligible catalogs within 60 seconds

**Constraints**: OData V4 only; supplier data remains canonical; no persisted product copies in
retailer catalogs; one catalog per retailer; up to 10 images per product, each up to 10 MB and only
JPEG, PNG, or WebP; optimistic concurrency must return a conflict for stale product writes; public
responses must be read-only and filtered server-side

**Scale/Scope**: At least 50 suppliers, 5,000 supplier products, 200 entries per retailer catalog,
and 100 concurrent public consumers; supplier and retailer account provisioning remains external to
this feature

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-checked after Phase 1 design.*

### Pre-Research Gate

| Constitutional gate | Design evidence | Status |
|---------------------|-----------------|--------|
| Supplier is the product source of truth; retailers do not recatalog or copy | `SupplierProduct` owns intrinsic fields; `CatalogEntry` stores only an association and retailer visibility | PASS |
| Supplier updates propagate while retailer values remain intact | Service reads resolve current associated product data; visibility stays on `CatalogEntry` | PASS |
| MVP simplicity and exclusions | One CAP service process, generic providers, metadata-driven admin UIs, and no commerce, AI, or advanced analytics | PASS |
| CAP Node.js/CDS and standard directories | Domain goes in `db/`, service contracts/rules in `srv/`, and all UIs in `app/` | PASS |
| SAPUI5 administration and optional React public catalog | Both administrative apps use SAP Fiori elements; React is isolated to the consumer app | PASS |
| OData V4 only | Three use-case-specific CAP services expose only OData V4; no V2 adapter is planned | PASS |
| Separation, reuse, and English identifiers | Shared domain and rule modules serve role-specific projections; code-level names are English | PASS |
| Validation and error handling | CDS constraints cover simple invariants; handlers cover ownership, state, image limits, uniqueness, and transactional errors | PASS |
| Preserve and inspect existing functionality | Repository inspection found only the CAP scaffold and documentation; the plan adds the first application feature without replacing existing behavior | PASS |
| Simple, readable design | Managed associations, compositions for owned children, generic CRUD, and limited custom handlers avoid premature layers | PASS |
| Feature compliance and architectural justification | The spec, this gate, and `research.md` document every material choice before implementation | PASS |

### Post-Design Gate

The Phase 1 design remains compliant. The data model contains no retailer-owned copy of supplier
fields; contracts separate supplier, retailer, and anonymous access; media and concurrency use
documented CAP facilities; and the project tree follows `db/`, `srv/`, and `app/`. No constitutional
exception or complexity waiver is required.

## Project Structure

### Documentation (this feature)

```text
specs/001-integrated-product-catalog/
├── plan.md
├── research.md
├── data-model.md
├── quickstart.md
├── contracts/
│   └── odata-v4.md
├── checklists/
│   └── requirements.md
└── tasks.md                 # Created later by $speckit-tasks
```

### Source Code (repository root)

```text
package.json
package-lock.json
xs-security.json
mta.yaml

db/
├── schema.cds
└── data/
    └── moveleira-Categories.csv

srv/
├── supplier-admin-service.cds
├── supplier-admin-service.js
├── retailer-admin-service.cds
├── retailer-admin-service.js
├── public-catalog-service.cds
├── public-catalog-service.js
├── catalog-rules.js
├── media-rules.js
├── annotations/
│   ├── supplier-admin.cds
│   └── retailer-admin.cds
└── i18n/
    ├── i18n.properties
    └── i18n_pt_BR.properties

app/
├── supplier-admin/
│   ├── ui5.yaml
│   ├── package.json
│   └── webapp/
│       ├── Component.js
│       ├── manifest.json
│       ├── ext/
│       ├── i18n/
│       └── test/
├── retailer-admin/
│   ├── ui5.yaml
│   ├── package.json
│   └── webapp/
│       ├── Component.js
│       ├── manifest.json
│       ├── ext/
│       ├── i18n/
│       └── test/
└── public-catalog/
    ├── package.json
    ├── vite.config.ts
    ├── index.html
    └── src/
        ├── main.tsx
        ├── routes/
        ├── components/
        ├── services/
        ├── test/
        └── i18n/

test/
├── unit/
├── integration/
├── contract/
└── e2e/
```

**Structure Decision**: Retain the native CAP single-project layout. Persistence and canonical
entities live in `db/`; role-specific contracts and business invariants live in `srv/`; each user
experience is an isolated app below `app/`. Shared business behavior is implemented once in CAP,
while each frontend owns only presentation and interaction concerns.

## Complexity Tracking

No constitution violations require tracking. The additional frontend technology is explicitly
permitted for the public catalog and does not duplicate backend business rules.
