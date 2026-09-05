<!--
Sync Impact Report
- Version change: unversioned template -> 1.0.0
- Modified principles: none (initial ratification)
- Added principles:
  - I. Supplier-Owned Product Truth
  - II. Reuse Without Manual Recataloging
  - III. Relationships Over Independent Copies
  - IV. Controlled Propagation and Retailer Overrides
  - V. MVP Validation First
  - VI. MVP Scope Discipline
  - VII. SAP CAP Backend
  - VIII. SAPUI5 Administrative Applications
  - IX. React as an Option for the Public Catalog
  - X. Standard SAP CAP Project Structure
  - XI. OData V4 Service Contracts
  - XII. Separation of Responsibilities
  - XIII. Avoid Code Duplication
  - XIV. English Code Identifiers
  - XV. Business-Rule Validation and Error Handling
  - XVI. Preserve Existing Functionality
  - XVII. Inspect Before Modifying
  - XVIII. Simple and Readable Code
  - XIX. Constitution Compliance for Every Feature
  - XX. Prior Justification for Architectural Changes
- Added sections:
  - MVP Scope and Acceptance Criteria
  - Development Workflow and Quality Gates
  - Governance
- Removed sections: none
- Follow-up TODOs: none
-->

# Moveleira Constitution

Moveleira is a digital platform that connects furniture-industry suppliers, retailers,
and consumers through integrated digital catalogs.

## Core Principles

### I. Supplier-Owned Product Truth

The supplier's product record MUST be the authoritative source for intrinsic product data,
including identity, description, specifications, dimensions, and media. Retailer catalogs MUST
derive this information from that record rather than establish a competing source of truth.
This preserves provenance and consistency throughout the catalog network.

### II. Reuse Without Manual Recataloging

Retailers MUST be able to discover and add supplier products without manually re-entering data
already maintained by the supplier. Retailer input MUST be limited to retailer-owned settings or
explicitly customizable fields. This removes redundant work and is central to the problem the
platform exists to solve.

### III. Relationships Over Independent Copies

A product added to a retailer catalog MUST retain a stable relationship to its original supplier
product. An independent product copy MUST NOT be the default implementation; any unavoidable
snapshot or duplication MUST have a documented requirement and retain provenance to the source.
This keeps shared data synchronized and prevents catalog drift.

### IV. Controlled Propagation and Retailer Overrides

Supplier changes MUST be capable of propagating to related retailer catalog entries. Fields that
retailers may customize MUST be explicitly modeled, and propagation MUST NOT overwrite active
retailer-owned values. The applicable propagation and override rules MUST be deterministic and
covered by validation. This balances supplier authority with retailer presentation needs.

### V. MVP Validation First

MVP decisions MUST optimize for simplicity, delivery speed, and validation of the integrated
catalog problem. Work that does not materially enable or validate the supplier-to-retailer-to-
consumer flow MUST be deferred. This keeps effort focused on the project's primary uncertainty.

### VI. MVP Scope Discipline

Marketplace functionality, payments, orders, artificial intelligence, and advanced analytics
MUST NOT be implemented before the complete MVP flow satisfies the acceptance criteria in this
constitution. Adding any of these capabilities earlier requires a constitution amendment. This
prevents premature expansion from delaying validation.

### VII. SAP CAP Backend

Backend services and persistence models MUST use SAP Cloud Application Programming Model with
Node.js and CDS. A deviation is an architectural change and MUST follow the prior-justification
rule. This standardizes backend implementation and reduces technology fragmentation.

### VIII. SAPUI5 Administrative Applications

Administrative interfaces used by suppliers and retailers MUST use SAPUI5. Shared administrative
behaviors MUST follow SAPUI5 conventions and consume the platform service contracts. This keeps
the operational experience aligned with the chosen SAP stack.

### IX. React as an Option for the Public Catalog

The public consumer catalog MAY use React when it supports the simplest viable delivery of the
public experience. Choosing React MUST NOT duplicate backend business rules in the client or
change the authoritative-source model. This permits an appropriate public frontend without
weakening platform boundaries.

### X. Standard SAP CAP Project Structure

The repository MUST follow the standard SAP CAP structure: `db/` for entities and persistence,
`srv/` for services and business rules, and `app/` for applications. New files MUST be placed in
the directory matching their responsibility. This makes ownership and runtime boundaries clear.

### XI. OData V4 Service Contracts

Application services MUST be exposed through OData V4 unless an approved architectural change
documents why another contract is required. Consumers MUST use published service contracts rather
than bypass service boundaries. This provides a consistent integration model across applications.

### XII. Separation of Responsibilities

Data modeling, service exposure, business rules, and user-interface concerns MUST remain in their
respective layers. UI code MUST NOT become the sole implementation of business invariants, and
persistence models MUST NOT absorb presentation concerns. This supports maintainability and
consistent enforcement across clients.

### XIII. Avoid Code Duplication

Business logic and reusable behavior MUST have one clear implementation when they represent the
same rule. Duplication MUST be removed when it creates competing behavior or maintenance risk;
shared abstractions MUST be introduced only when the reuse is concrete. This avoids drift without
encouraging speculative abstraction.

### XIV. English Code Identifiers

Entity, service, function, variable, and other code-level identifiers MUST be written in English.
User-facing language MAY follow product localization requirements. This ensures a consistent
technical vocabulary while preserving flexibility in the interface.

### XV. Business-Rule Validation and Error Handling

Every material business rule MUST have explicit validation and predictable error handling at the
appropriate service boundary. Failures MUST return actionable errors and MUST NOT leave partial or
invalid state. Tests MUST cover the successful path and relevant rejected paths. This makes core
catalog behavior dependable and diagnosable.

### XVI. Preserve Existing Functionality

Existing behavior MUST NOT be changed unless the feature or defect being addressed requires it.
Any intentional behavior change MUST be identified in the relevant specification or plan and
verified against affected flows. This limits regressions and keeps changes reviewable.

### XVII. Inspect Before Modifying

Before changing an existing file, the implementer MUST inspect the current project structure,
nearby conventions, and affected dependencies. The resulting change MUST fit established patterns
unless a justified architectural change supersedes them. This prevents changes based on false
assumptions about the codebase.

### XVIII. Simple and Readable Code

Implementations MUST favor the simplest readable design that meets current, verified requirements.
Premature abstractions, unnecessary indirection, and speculative extensibility MUST be rejected.
This shortens delivery time and makes the MVP easier to evolve from evidence.

### XIX. Constitution Compliance for Every Feature

Every new feature specification, plan, task set, and implementation review MUST demonstrate
compliance with this constitution. Any conflict MUST be resolved before implementation proceeds,
either by changing the feature or formally amending the constitution. This turns the principles
into enforceable delivery gates.

### XX. Prior Justification for Architectural Changes

Any material architectural change MUST be documented and justified before implementation. The
justification MUST state the problem, the proposed decision, relevant alternatives, tradeoffs, and
migration or compatibility impact. Approval MUST occur before code depending on the decision is
merged. This keeps foundational decisions deliberate and auditable.

## MVP Scope and Acceptance Criteria

The MVP is complete only when all of the following capabilities work as one end-to-end flow:

- A supplier can create, view, update, and manage its products as the authoritative owner.
- A retailer can view available suppliers and their products without recataloging them.
- A retailer can add a supplier product to its catalog through a persistent relationship to the
  original product.
- A retailer can manage which related products are visible in its own store catalog.
- A consumer can access a public catalog and view the products made visible by the retailer.

An implementation that provides isolated screens or records without preserving the linked product
flow does not satisfy the MVP. Features excluded by Principle VI remain out of scope until every
criterion above is demonstrably functional.

## Development Workflow and Quality Gates

- Before implementation, the responsible contributor MUST identify the affected data model,
  services, business rules, interfaces, and existing behavior.
- Feature specifications and plans MUST show how supplier ownership, retailer relationships,
  propagation, and retailer overrides are preserved wherever those concerns apply.
- Reviews MUST verify CAP directory placement, OData V4 contracts, layer separation, English code
  identifiers, validation, error handling, and regression risk.
- Tests MUST exercise material business rules and the integration boundaries they cross. Changes
  to linked product behavior MUST include coverage of supplier updates and retailer overrides.
- The smallest complete change that advances an MVP acceptance criterion MUST take precedence over
  unrelated infrastructure, abstraction, or out-of-scope capability work.
- Architectural decisions MUST be approved and documented before dependent implementation begins.

## Governance

This constitution is the highest-priority project governance document. Specifications, plans,
tasks, implementation, and reviews MUST conform to it. When another project practice conflicts
with this document, this constitution prevails.

Amendments MUST be proposed in writing and include the rationale, affected principles or sections,
compatibility impact, and any required migration. Project maintainers MUST approve an amendment
before it is merged or used to authorize dependent implementation. Every approved amendment MUST
update the version, last-amended date, and Sync Impact Report.

Versions follow semantic versioning for governance: MAJOR for removal or incompatible redefinition
of principles; MINOR for a new principle, section, or material expansion; PATCH for clarifications
that do not alter obligations. The ratification date remains the original adoption date.

Every feature review and release-readiness review MUST include an explicit constitution compliance
check. Noncompliance blocks approval until the work is corrected or this constitution is amended.
Complexity and exceptions MUST be justified in the governing specification or plan and reviewed
before implementation.

**Version**: 1.0.0 | **Ratified**: 2026-09-05 | **Last Amended**: 2026-09-05
