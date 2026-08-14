# Exact Catalog Model Relation Pairs

## Goal

Preserve the exact source- and target-kind pairs declared by catalog model
layers, so that the catalog processor only generates inverse relations for
declared pairs. At the same time, make the existing `AiResource` model
self-contained by registering every relation pair used by its current fields.

## Scope

This change will:

- retain exact kind-pair information while compiling relation operations;
- make `getRelations({ kind })` expose only the target kinds declared for that
  source kind;
- keep `listRelations()` as a union-style summary of every source and target
  kind for a relation type;
- register the existing `AiResource` ownership, system, and dependency pairs;
- restrict the existing `AiResource` system and dependency fields to their
  intended target kinds; and
- add compilation and processing regression coverage.

This change will not:

- register the proposed plugin or marketplace `AiResource` hierarchy;
- change relation metadata replacement or amendment semantics;
- change kind-removal behavior; or
- redesign the public catalog model API.

## Compatibility

No public TypeScript interface or serialized opaque type changes. In
particular, `CatalogModel`, `CatalogModelRelation`, `CatalogModelRelationSummary`,
`OpaqueCatalogModelLayer`, `declareRelation.v1`, and `updateRelation.v1` retain
their current shapes and revisions.

The existing relation operations already carry one exact `fromKind` and
`toKind`. The compiler currently loses that correlation by merging them into
independent sets. The fix is therefore entirely within compiled state and does
not require a new opaque type revision. Previously authored model layers remain
readable without migration.

The observable behavior change is intentional: inverse relations will no
longer be generated for source/target combinations that were never declared.

## Compiler Design

Each technical relation type will continue to have one metadata record for its
description, forward title, reverse type, and reverse title. Instead of storing
independent source- and target-kind sets, that record will store a mapping from
each source kind to its exact set of target kinds.

Both declaration and update operations add their exact source/target pair to
that mapping. Adding or updating one pair does not remove existing pairs.
Existing metadata update behavior remains unchanged and out of scope.

Compilation produces two views:

- `listRelations()` derives the existing union-style summary from every exact
  pair, preserving its current shape and one-summary-per-relation-type behavior.
- `getRelations({ kind })` returns the relation with only the requested source
  kind and that source kind's exact declared targets.

The model processor already consumes `getRelations({ kind })`. Its current
membership test will therefore become exact without changing the processor or
the public relation type.

## AiResource Completeness

The `AiResource` layer will update the well-known pairs used by its existing
relation fields:

- `AiResource ownedBy Group/User`, reversed by `ownerOf`;
- `AiResource partOf System`, reversed by `hasPart`; and
- `AiResource dependsOn AiResource`, reversed by `dependencyOf`.

The existing `spec.system` field will allow only `System` targets, and the
existing skill `spec.dependsOn` field will allow only `AiResource` targets.
The ownership field already restricts targets to `Group` and `User`.

No `AiResource partOf AiResource` pair is added. That pair belongs to the
pending plugin and marketplace changes that introduce fields requiring it.

## Testing

Compiler tests will prove that:

- two disjoint declarations of one relation type remain two exact
  source/target possibilities;
- `updateRelationPair` adds an exact pair without replacing existing pairs;
- the reverse relation receives the corresponding exact pair; and
- `listRelations()` continues returning union summaries.

Processor coverage will prove that a declared pair generates its inverse while
an undeclared cross-product combination does not.

AiResource coverage will compile the default and AiResource layers together
and verify the exact ownership, system, and dependency targets, including their
reverse mappings.

## Release Notes

The `@backstage/catalog-model` package will receive a patch changeset describing
the corrected inverse-relation generation and complete relation registration
for existing `AiResource` fields.
