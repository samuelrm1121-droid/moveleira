# Data Model: Integrated Product Catalog

## Modeling Conventions

- Namespace and all code-level identifiers use English.
- Persistent business entities use CAP `cuid` UUID keys and `managed` audit fields.
- `SupplierProduct.modifiedAt` is the aggregate ETag used for optimistic concurrency.
- Associations represent independently owned records; compositions represent children that cannot
  exist without their parent.
- No retailer entity stores a copy of supplier-owned product fields.
- Simple input constraints are declarative; cross-record ownership, state, image-count, and
  effective-visibility rules are enforced in service handlers and transactions.
- Dimensions are stored as positive decimal centimeters for the MVP.

## Ownership Matrix

| Data | Owner | Writable through | Publicly exposed |
|------|-------|------------------|------------------|
| Supplier identity and name | Platform provisioning | Outside this feature | Name only when a product is public |
| Retailer identity, name, and public slug | Platform provisioning | Outside this feature | Name and slug |
| Category reference data | Platform | Seed/configuration only | Code and name when used |
| Product intrinsic fields and activity | Supplier | Supplier admin service | Only through an eligible catalog entry |
| Product image content and order | Supplier | Supplier admin service | Only through an eligible catalog entry |
| Catalog membership and visibility | Retailer | Retailer admin service | Only when effectively visible |

## Entities

### Supplier

Represents a pre-provisioned supplier organization.

| Field | Type | Required | Rules |
|-------|------|----------|-------|
| `ID` | UUID | Yes | Canonical immutable key |
| `name` | String(120) | Yes | Non-empty display name |
| `isActive` | Boolean | Yes | Defaults to `true`; inactive suppliers are unavailable to retailers |
| managed fields | Timestamp/User | Yes | Created and modified metadata supplied by CAP |

Relationships:

- One supplier has many `SupplierProduct` records through an unmanaged reverse association.
- Administrative users are scoped to a supplier by trusted identity context; user provisioning is
  outside this data model.

### Retailer

Represents a pre-provisioned retailer organization.

| Field | Type | Required | Rules |
|-------|------|----------|-------|
| `ID` | UUID | Yes | Canonical immutable key |
| `name` | String(120) | Yes | Non-empty public store name |
| `publicSlug` | String(80) | Yes | Globally unique, stable, URL-safe identifier |
| `isActive` | Boolean | Yes | Defaults to `true`; inactive stores have no public catalog response |
| managed fields | Timestamp/User | Yes | Created and modified metadata supplied by CAP |

Relationships:

- One retailer has exactly one `RetailerCatalog` through an explicit association.
- A unique constraint on `Retailer.publicSlug` prevents ambiguous public routes.
- Retailer provisioning must create the catalog in the same transaction; fixtures do the same.

### Category

Central reference classification shared by every supplier.

| Field | Type | Required | Rules |
|-------|------|----------|-------|
| `ID` | UUID | Yes | Canonical immutable key |
| `code` | String(40) | Yes | Unique, stable, English machine identifier |
| `name` | String(100) | Yes | Non-empty user-facing label |
| `isActive` | Boolean | Yes | Defaults to `true`; only active categories may be newly selected |
| `sortOrder` | Integer | Yes | Non-negative central display order |

Relationships and rules:

- One category can classify many supplier products.
- Categories are loaded from `db/data/moveleira-Categories.csv` for the MVP.
- Deactivating reference data does not erase existing product associations. A supplier cannot assign
  an inactive category to a new product or change a product to one.

### SupplierProduct

Canonical supplier-owned product. Every consumer-facing product value originates here.

| Field | Type | Required | Rules |
|-------|------|----------|-------|
| `ID` | UUID | Yes | Canonical immutable key |
| `supplier` | Association to Supplier | Yes | Immutable after creation; must match authenticated supplier |
| `category` | Association to Category | Yes | Target must exist and be active when selected |
| `name` | String(160) | Yes | Trimmed and non-empty |
| `description` | LargeString | No | Supplier-owned rich/plain description accepted as text |
| `supplierReference` | String(80) | No | Supplier's reference; not globally unique in the MVP |
| `brand` | String(120) | No | Supplier-owned value |
| `model` | String(120) | No | Supplier-owned value |
| `material` | String(160) | No | Supplier-owned value |
| `widthCm` | Decimal(9,2) | No | Greater than zero when supplied |
| `heightCm` | Decimal(9,2) | No | Greater than zero when supplied |
| `depthCm` | Decimal(9,2) | No | Greater than zero when supplied |
| `isActive` | Boolean | Yes | Defaults to `true` |
| managed fields | Timestamp/User | Yes | `modifiedAt` is annotated as the OData ETag |

Relationships:

- Belongs to exactly one supplier and one category through managed associations.
- Composes zero to ten `ProductImage` children.
- Is referenced by zero or more `CatalogEntry` records. This direction is an association, never a
  composition, so retailer lifecycle operations cannot alter or delete the source product.

Validation and concurrency:

- Only users scoped to the owning supplier can create, read, or update the product.
- DELETE is not exposed in the MVP; activity controls availability.
- PATCH requires the current ETag. A stale ETag returns HTTP 412 without overwriting confirmed data.
- Creating, deleting, replacing, or reordering an image also advances the product aggregate ETag.

### ProductImage

Attachment-based image child whose content is maintained by the platform.

| Field | Type | Required | Rules |
|-------|------|----------|-------|
| `ID` | UUID | Yes | Canonical immutable key |
| `product` | Association to SupplierProduct | Yes | Composition parent; immutable |
| `fileName` | String(255) | Yes | Sanitized display filename |
| `mediaType` | String(100) | Yes | `image/jpeg`, `image/png`, or `image/webp` |
| `size` | Integer64 | Yes | 1 to 10,485,760 bytes |
| `content` | Attachment/media stream | Yes | Managed by `@cap-js/attachments`, not embedded in list payloads |
| `position` | Integer | Yes | 1 through 10; unique inside the product |
| managed fields | Timestamp/User | Yes | Attachment lifecycle metadata |

Relationships and invariants:

- A product composes at most ten images.
- The pair (`product`, `position`) is unique.
- An attachment without successfully stored, accepted content is never returned by the public
  service; failed uploads are rolled back or cleaned up.
- Deleting the image record deletes its managed content, but deleting products remains unavailable.
- Public media reads re-check that the parent product has an effectively visible catalog entry.

### RetailerCatalog

The single catalog owned by a retailer.

| Field | Type | Required | Rules |
|-------|------|----------|-------|
| `ID` | UUID | Yes | Canonical immutable key |
| `retailer` | Association to Retailer | Yes | Immutable; unique across catalogs |
| managed fields | Timestamp/User | Yes | Created and modified metadata supplied by CAP |

Relationships and invariants:

- Composes zero or more `CatalogEntry` children.
- Unique constraint on `retailer` guarantees at most one catalog per retailer; the provisioning
  transaction guarantees existence.
- The retailer admin service always scopes the catalog to the authenticated retailer.

### CatalogEntry

Persistent link between a retailer catalog and a canonical supplier product.

| Field | Type | Required | Rules |
|-------|------|----------|-------|
| `ID` | UUID | Yes | Canonical immutable key |
| `catalog` | Association to RetailerCatalog | Yes | Composition parent; derived from current retailer |
| `product` | Association to SupplierProduct | Yes | Immutable canonical source reference |
| `isVisible` | Boolean | Yes | Defaults to `false`; retailer-owned |
| managed fields | Timestamp/User | Yes | `modifiedAt` may be used as an ETag for visibility edits |

Relationships and invariants:

- The pair (`catalog`, `product`) is unique, preventing duplicate additions.
- Creation is performed by the `addToCatalog` service action so the client cannot choose another
  retailer's catalog or an inactive source product.
- The entry contains no supplier-owned product name, description, category, dimensions, or media.
- DELETE is not exposed in the MVP; hiding the entry preserves the relationship.

## State Transitions

### Supplier Product Activity

```text
create valid product
        │
        ▼
      ACTIVE ── deactivate ──> INACTIVE
        ▲                           │
        └──────── reactivate ───────┘
```

- `ACTIVE`: discoverable and eligible for public display when its catalog entry is visible.
- `INACTIVE`: cannot be newly added and is not publicly displayed; existing entries remain linked.
- Reactivation reuses every existing catalog entry and its retailer-owned visibility choice.

### Catalog Visibility

```text
addToCatalog
     │
     ▼
   HIDDEN ── publish ──> VISIBLE_REQUESTED
     ▲                         │
     └──────── hide ───────────┘
```

`VISIBLE_REQUESTED` is the retailer's stored preference. Effective public visibility is derived:

```text
catalogEntry.isVisible
AND catalogEntry.product.isActive
AND catalogEntry.product.supplier.isActive
AND catalogEntry.catalog.retailer.isActive
```

No transition copies or snapshots product data.

## Integrity and Indexes

Required uniqueness constraints:

- `Retailer.publicSlug`
- `Category.code`
- `RetailerCatalog.retailer`
- (`CatalogEntry.catalog`, `CatalogEntry.product`)
- (`ProductImage.product`, `ProductImage.position`)

Required query indexes after measuring generated database plans:

- `SupplierProduct(supplier, isActive)` for supplier ownership and retailer browsing.
- `CatalogEntry(catalog, isVisible)` for public catalog reads.
- `CatalogEntry(product)` for activity and media eligibility checks.

Database referential integrity is enabled for persisted associations. Friendly target and state
validation still runs at the service boundary so callers do not receive raw database errors.

## Expected Volume

| Record/content | MVP validation target |
|----------------|-----------------------|
| Suppliers | 50 or more |
| Supplier products | 5,000 or more |
| Entries per retailer catalog | 200 or more |
| Images per product | 0 to 10 |
| Image size | Up to 10 MB |
| Concurrent public consumers | 100 |
