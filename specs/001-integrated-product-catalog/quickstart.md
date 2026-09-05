# Quickstart: Integrated Product Catalog

## Purpose

This guide defines the expected local setup and the minimum validation journey for the integrated product catalog MVP. The commands and application routes below become available as the implementation tasks create the project scripts and applications described in the implementation plan.

## Prerequisites

- Node.js 24 LTS
- npm 12 or a compatible npm release
- A modern browser
- Local ports `4004` and `5173` available

Development and automated tests use SQLite. Production uses SAP HANA Cloud and production authentication; neither is required for the local MVP validation.

## Install and Start

From the repository root:

```bash
npm install
npm run dev
```

The development command is expected to start:

- CAP services and SAPUI5 applications at `http://localhost:4004`
- the React public catalog at `http://localhost:5173`

Expected application routes:

- Supplier administration: `http://localhost:4004/supplier-admin/`
- Retailer administration: `http://localhost:4004/retailer-admin/`
- Public catalog: `http://localhost:5173/catalog/{retailerSlug}`

Use the mock supplier and retailer identities supplied by the development configuration. Mock authentication must never be enabled in production.

## Prepare Local Data

Deploy the CDS model and load development seed data:

```bash
npm run db:deploy
```

The seed data should provide at least:

- two suppliers, owned by different mock supplier users;
- two retailers, owned by different mock retailer users;
- an initial category list;
- one retailer catalog with a public slug.

Seeded supplier products are optional because creating a product is part of the primary validation journey.

## Automated Validation

Run the complete quality gate:

```bash
npm run lint
npm run typecheck
npm test
npm run build
```

When individual suites are needed during development:

```bash
npm run test:backend
npm run test:ui5
npm run test:public
npm run test:e2e
```

The test suites must cover the service contract matrix in [contracts/odata-v4.md](contracts/odata-v4.md), including authorization, duplicate prevention, optimistic concurrency, visibility, propagation, and public read-only behavior.

## Primary MVP Validation Journey

### 1. Supplier creates and publishes a product

1. Sign in as a supplier user.
2. Open the supplier administration application.
3. Create a product with a name, description, category, basic product information, and at least one supported image.
4. Activate the product.
5. Confirm that it appears in the supplier's own product list.

Expected result: the product is stored once under the authenticated supplier and exposes an ETag for safe updates.

### 2. Retailer discovers and links the product

1. Sign in as a retailer user.
2. Open the retailer administration application.
3. Find the supplier and open its active product list.
4. Add the product to the retailer catalog.
5. Confirm that the new catalog entry starts hidden from the public catalog.
6. Try to add the same product again.

Expected result: the first operation creates a persistent relationship to the supplier product; it does not copy the product data. The duplicate attempt is rejected with `409 Conflict`.

### 3. Retailer controls public visibility

1. Mark the catalog entry as visible.
2. Open the public catalog using the retailer's slug.

Expected result: the consumer can see the product information and images. Entries marked hidden and inactive supplier products are absent.

### 4. Supplier changes propagate

1. Keep the public catalog open.
2. As the supplier, edit the product name or description and save it.
3. Refresh or revisit the public catalog within 60 seconds.

Expected result: the updated supplier information is shown without recreating or relinking the retailer entry. The retailer's visibility choice remains unchanged.

### 5. Supplier deactivates and reactivates the product

1. Deactivate the supplier product.
2. Verify that it is no longer shown publicly, even though the retailer entry remains visible.
3. Reactivate it.

Expected result: effective public visibility follows both the supplier product state and the retailer entry state. Reactivation restores the product without a new link.

## Safety and Isolation Checks

- A supplier cannot read or change another supplier's private administration data.
- A retailer cannot administer another retailer's catalog.
- A consumer can only use the explicitly public, read-only service.
- An update using an obsolete ETag receives `412 Precondition Failed` and does not overwrite newer data.
- Unsupported image formats, oversized files, and more than ten images per product are rejected with actionable errors.
- Removing a supplier product that is already linked must follow the service's referential-integrity policy; normal lifecycle control uses activation and deactivation.

## Acceptance Outcome

The MVP flow is validated when a supplier-created product can be found and linked by a retailer, remains linked to its source, respects retailer visibility, reflects subsequent supplier changes, and is visible to consumers only when both source and catalog-entry states allow it.

For the planned entities and lifecycle rules, see [data-model.md](data-model.md). For service paths, actions, errors, and authorization expectations, see [contracts/odata-v4.md](contracts/odata-v4.md).
