# OData V4 Contract: Integrated Product Catalog

## Contract Scope

The feature exposes three use-case-specific OData V4 services under `/odata/v4`. Entity and field
names are English. Administrative services require identity and role authorization; the public
service is explicitly anonymous and read-only.

The CDS metadata generated during implementation is the machine-readable contract. This document
defines the stable resource surface, invariants, and expected HTTP behavior that contract tests must
enforce.

## Shared Protocol Rules

- OData V4 is the only supported service protocol; no OData V2 adapter is enabled.
- JSON responses use standard OData envelopes, including `value`, `@odata.etag`, and opaque
  `@odata.nextLink` values where applicable.
- UUID keys are opaque and must not encode organization or sequence meaning.
- Administrative writes use standard OData change-set transactions where several writes must be
  atomic.
- Validation errors use the OData error envelope with a stable error code and actionable message.
- Server-side authorization and eligibility filters apply even if the client omits or alters query
  filters.
- Collection responses use a default page size of 20 and a hard maximum of 100. Clients follow
  `@odata.nextLink` without reconstructing it.
- Product PATCH operations require `If-Match` with the last observed ETag. `If-Match: *` is not used
  by the administrative clients.

## Roles and Data Scope

| Service | Required role | Organization scope | Writes |
|---------|---------------|--------------------|--------|
| `SupplierAdminService` | `Supplier` | Current supplier only | Product and owned image create/update; image delete |
| `RetailerAdminService` | `Retailer` | Current retailer catalog only | Add relationship and change visibility |
| `PublicCatalogService` | `any` | Published data selected by catalog slug | None |

The trusted identity context supplies the organization identifier for administrative requests. A
client-provided supplier, retailer, or catalog key never overrides this scope.

## SupplierAdminService

**Base path**: `/odata/v4/supplier-admin`

### Entity Sets

| Entity set | Allowed operations | Contract |
|------------|--------------------|----------|
| `Products` | GET, POST, PATCH | Only products owned by the current supplier; DELETE rejected |
| `Categories` | GET | Active central categories for value help; always read-only |
| `ProductImages` | GET, POST, PATCH, DELETE | Only images below products owned by the current supplier |

### Products

Public service fields are not implied by this administrative shape. The supplier projection exposes:

- `ID`, `name`, `description`, `supplierReference`, `brand`, `model`, `material`
- `widthCm`, `heightCm`, `depthCm`, `isActive`
- `category` association and `images` composition
- managed timestamps and `@odata.etag`

#### Create

```http
POST /odata/v4/supplier-admin/Products
Content-Type: application/json

{
  "name": "Mesa Aurora",
  "category_ID": "<active-category-uuid>",
  "description": "Mesa de jantar",
  "material": "Madeira",
  "widthCm": 180,
  "heightCm": 78,
  "depthCm": 90
}
```

Expected behavior:

- The server derives `supplier` from the authenticated identity.
- `isActive` defaults to `true`.
- Success returns 201, the created resource, and its ETag.
- A missing/blank name, inactive/missing category, or non-positive dimension returns 400 without a
  partial product.

#### Read and list

```http
GET /odata/v4/supplier-admin/Products?$select=ID,name,isActive,modifiedAt&$expand=category
GET /odata/v4/supplier-admin/Products(<product-id>)?$expand=category,images
```

Another supplier's product is not returned. Direct lookup outside the caller's scope returns 404 to
avoid confirming the resource exists.

#### Update, activate, or deactivate

```http
PATCH /odata/v4/supplier-admin/Products(<product-id>)
If-Match: W/"<etag>"
Content-Type: application/json

{ "isActive": false }
```

Expected behavior:

- Success returns 200 or 204 and a new ETag on the next representation.
- A stale ETag returns 412 `PRODUCT_CONFLICT`; confirmed data remains unchanged.
- Missing `If-Match` returns 428 `ETAG_REQUIRED`.
- Attempts to change `supplier_ID`, `ID`, or managed fields return 400.
- DELETE and permanent product removal return 405.

### Product Images

Create metadata beneath an owned product, then upload the file content using the returned image ID:

```http
POST /odata/v4/supplier-admin/ProductImages
Content-Type: application/json

{
  "product_ID": "<owned-product-uuid>",
  "fileName": "mesa-aurora.webp",
  "mediaType": "image/webp",
  "position": 1
}
```

```http
PUT /odata/v4/supplier-admin/ProductImages(<image-id>)/content
If-Match: W/"<current-product-etag>"
Content-Type: image/webp
Content-Length: <bytes-at-most-10485760>

<binary image body>
```

Expected behavior:

- Accepted types are `image/jpeg`, `image/png`, and `image/webp`.
- Each file is at most 10,485,760 bytes and each product has at most ten images.
- `position` is 1 through 10 and unique within the product.
- Unsupported type returns 415; excessive body size returns 413; count or position conflicts return
  409; unauthorized parent product returns 404.
- Failed content storage must not leave a publicly readable metadata-only record.
- A successful create, replace, delete, or reorder advances the parent product ETag.
- `GET /ProductImages(<image-id>)/content` streams the current file to its owning supplier.
- DELETE removes only the image and its managed content, never the product.

## RetailerAdminService

**Base path**: `/odata/v4/retailer-admin`

### Entity Sets

| Entity set | Allowed operations | Contract |
|------------|--------------------|----------|
| `Suppliers` | GET | Active suppliers available to the retailer |
| `Products` | GET and bound `addToCatalog` action | Active canonical products; supplier fields read-only |
| `Catalog` | GET | Exactly one record for the current retailer |
| `CatalogEntries` | GET, PATCH | Current retailer entries; only `isVisible` may be changed |

### Browse suppliers and products

```http
GET /odata/v4/retailer-admin/Suppliers?$select=ID,name&$orderby=name
GET /odata/v4/retailer-admin/Products?$filter=supplier_ID eq <supplier-id>&$expand=category,images
```

The service returns active suppliers and active products only. Reads resolve current supplier-owned
fields from `SupplierProduct`; catalog entries do not supply duplicated values.

### Add a product to the current catalog

The `addToCatalog` action is bound to a product selected in the retailer app:

```http
POST /odata/v4/retailer-admin/Products(<product-id>)/RetailerAdminService.addToCatalog
Content-Type: application/json

{}
```

Expected behavior:

- The server resolves the current retailer's single catalog.
- The product must still be active when the transaction executes.
- Success creates one `CatalogEntry` with `isVisible = false` and returns 201 or 200 with the entry.
- Repeating the action for the same catalog/product returns 409 `PRODUCT_ALREADY_IN_CATALOG` and the
  existing entry identifier; no duplicate row is created.
- An inactive or missing product returns 409 `PRODUCT_NOT_AVAILABLE` or 404 respectively.

### Change retailer visibility

```http
PATCH /odata/v4/retailer-admin/CatalogEntries(<entry-id>)
If-Match: W/"<etag>"
Content-Type: application/json

{ "isVisible": true }
```

Only `isVisible` is writable. Attempts to modify the catalog, source product, source fields, ID, or
managed fields return 400. Access outside the current retailer returns 404. Hiding an entry preserves
the relationship, and DELETE returns 405.

The requested flag may remain `true` while the source product is inactive; effective public
visibility remains false until the supplier reactivates it.

## PublicCatalogService

**Base path**: `/odata/v4/public-catalog`

The service is annotated explicitly with `requires: 'any'` and `readonly`. It exposes narrow
projections, not persistence entities or administrative attachment services. POST, PUT, PATCH, and
DELETE return 405 for every public resource.

### Public resource shape

`Catalogs` is keyed by the retailer's stable `publicSlug` and exposes only:

- `slug`
- `retailerName`
- navigation `products`

Each `products` navigation row is dynamically derived from an eligible `CatalogEntry` and its
current `SupplierProduct`. It exposes:

- `ID` (catalog-entry identifier), `sourceProductID`
- `name`, `description`, `supplierReference`, `brand`, `model`, `material`
- `widthCm`, `heightCm`, `depthCm`
- `supplierName`, `categoryCode`, `categoryName`
- navigation `images` with `ID`, `fileName`, `mediaType`, `position`, and a relative media URL

It does not expose organization attributes, audit users, hidden entries, inactive products, binary
content in JSON, or writable service links.

### Read a catalog and products

```http
GET /odata/v4/public-catalog/Catalogs('<public-slug>')
GET /odata/v4/public-catalog/Catalogs('<public-slug>')/products?$orderby=name&$top=20
GET /odata/v4/public-catalog/Catalogs('<public-slug>')/products(<entry-id>)
```

Eligibility is always enforced by the server:

```text
entry.isVisible
AND entry.product.isActive
AND entry.product.supplier.isActive
AND entry.catalog.retailer.isActive
```

An unknown or inactive retailer slug returns 404. A valid catalog with no eligible products returns
200 and an empty `value` collection.

### Read public image content

```http
GET /odata/v4/public-catalog/Catalogs('<public-slug>')/products(<entry-id>)/images(<image-id>)/content
```

The handler validates the catalog, entry, product, and image relationship plus current eligibility
before streaming. Hidden or inactive ancestry returns 404. The response carries the stored media
type and inline filename; no caller-supplied external URL is followed.

### Cache and freshness

- Catalog JSON and public media responses use revalidation or a cache lifetime no longer than 30
  seconds.
- The React client treats data as stale after at most 30 seconds, refetches an open catalog at least
  once per 60 seconds, and refetches on window focus.
- Built frontend assets with content hashes may use long immutable caching; HTML remains revalidated.
- The server, not the browser, owns activation and visibility filtering on every uncached read.

## Standard Error Contract

| HTTP | Stable code | Meaning |
|------|-------------|---------|
| 400 | `VALIDATION_ERROR` | Missing, malformed, immutable, or invalid field |
| 401 | `AUTHENTICATION_REQUIRED` | Administrative request has no valid identity |
| 403 | `FORBIDDEN` | Identity lacks the required role |
| 404 | `NOT_FOUND` | Resource absent, outside caller scope, or not publicly eligible |
| 409 | `PRODUCT_ALREADY_IN_CATALOG` | Duplicate catalog relationship attempted |
| 409 | `PRODUCT_NOT_AVAILABLE` | Source product became inactive before addition |
| 409 | `IMAGE_LIMIT_REACHED` | Image count or position invariant failed |
| 412 | `PRODUCT_CONFLICT` | ETag no longer matches current product state |
| 413 | `IMAGE_TOO_LARGE` | Image exceeds 10 MB |
| 415 | `UNSUPPORTED_IMAGE_TYPE` | Image is not JPEG, PNG, or WebP |
| 428 | `ETAG_REQUIRED` | Update omitted required concurrency precondition |

All rejected writes leave product, image metadata/content, catalog relationship, and visibility state
unchanged unless the entire OData change set commits successfully.

## Contract Test Matrix

- Supplier A cannot read or mutate Supplier B's products or images.
- Retailer A cannot read or mutate Retailer B's catalog entries.
- A stale product ETag returns 412 and preserves the first confirmed update.
- Invalid category, dimensions, media type, size, image count, or order is rejected atomically.
- Duplicate `addToCatalog` calls produce one relationship and one stable 409 response.
- Public writes are rejected and public reads never return hidden/inactive products.
- Supplier product edits appear through existing retailer and public projections without copying.
- Deactivation removes public eligibility while preserving the entry and retailer visibility flag;
  reactivation restores eligibility when that flag remains true.
- Public media cannot be read through a catalog/product/image ancestry that is no longer eligible.
- Paging stays bounded and emitted `@odata.nextLink` values can be followed unchanged.
