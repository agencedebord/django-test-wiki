---
domain: lookups
confidence: llm-analyzed
last_updated: 2026-04-03
related_files:
  - docs/ref/models/lookups.txt
  - django/contrib/gis/db/models/lookups.py
  - django/db/models/lookups.py
---

# Lookups

## What this domain does
The `lookups` domain implements Django's query expression system for translating Python filter expressions into SQL WHERE clauses. It defines the two-layer abstraction that powers all ORM filtering: `Transform` (field-to-field conversions in a lookup chain) and `Lookup` (the terminal boolean comparison). Every `Model.objects.filter(field__lookup=value)` call ultimately resolves through classes defined or registered here. The domain is almost entirely consumed by other domains — it is infrastructure, not business logic. `[llm-analyzed]`

## Key behaviors
- **Bilateral transforms apply to both sides of a comparison**: When a `Transform` sets `bilateral=True`, it is applied to both the LHS (field) and RHS (filter value) symmetrically. The transforms appear in the same order on the RHS as in the lookup expression. This is used for case-insensitive lookups where the same normalization function must be applied to both sides. Bilateral transforms on nested QuerySets are explicitly forbidden and raise `NotImplementedError` immediately in `Lookup.__init__`. `[llm-analyzed]`
- **Lookups wrap nested lookups in parentheses to preserve operator precedence**: In `process_lhs`, if the LHS is itself a `Lookup` instance, its SQL is wrapped in parentheses. Similarly in `process_rhs`, any RHS expression that is not a raw `Value` gets parenthesized unless it already starts with '('. This prevents precedence bugs when composing lookups. `[llm-analyzed]`
- **Oracle backend gets special-cased for boolean expressions**: `Lookup.as_oracle` wraps any conditional expression (EXISTS, filters) in a `CASE WHEN ... THEN True ELSE False END` before comparison. This is required because Oracle does not allow boolean expressions to appear directly in WHERE clauses alongside other expressions. `select_format` applies the same wrapping for SELECT/GROUP BY contexts. `[llm-analyzed]`
- **RHS preparation is skipped for expressions that already know how to compile themselves**: In `get_prep_lookup`, if `self.rhs` already has a `resolve_expression` method (i.e. it is an ORM expression like `F()`, `Subquery`, etc.), the field's `get_prep_value` is bypassed entirely. Only raw Python values get coerced through the field's type system. This is the mechanism that allows filter values to be either plain Python objects or composed query expressions. `[llm-analyzed]`
- **GIS lookups delegate SQL generation to backend-specific SpatialOperator objects**: Unlike standard `BuiltinLookup` where `get_rhs_op` returns a string operator, `GISLookup.get_rhs_op` returns a `SpatialOperator` object with its own `as_sql()` method. This allows geometry operators (e.g. PostGIS `&&`, `~=`, `ST_Distance`) to produce arbitrarily complex SQL that may interleave LHS and RHS parts, which a simple string template cannot express. `[llm-analyzed]`

## Domain interactions
- **Db (database backends)**: Lookups call `connection.ops.get_geom_placeholder_sql`, `connection.ops.Adapter`, and `connection.ops.gis_operators` to produce backend-specific SQL. The `as_vendorname` dispatch mechanism (e.g. `as_oracle`, `as_postgresql`) allows lookups to emit different SQL per backend without subclassing. `[llm-analyzed]`
- **Expressions**: Lookup inherits from Expression and participates fully in the query expression tree. It calls `compiler.compile()` on sub-expressions and uses `Case/When/Value/Func` from expressions to build Oracle compatibility wrappers and bilateral transform chains. `[llm-analyzed]`
- **Fields**: Lookups are registered on field classes via `RegisterLookupMixin`. The field's `output_field` and `get_prep_value` determine how raw Python values are coerced for the RHS. Fields like IntegerField, UUIDField, and DateTimeField have specialized lookup subclasses that override default comparison behavior. `[llm-analyzed]`
- **Contrib.gis fields**: GIS lookups are registered directly on `BaseSpatialField` at module import time using the `@BaseSpatialField.register_lookup` decorator. All 30+ geometry and distance operators become available on spatial fields through this registration, with no opt-in required by model authors. `[llm-analyzed]`
- **All filtering domains (article, bar, base, constraints, etc.)**: Every domain that performs ORM filtering implicitly depends on this domain. The `__exact`, `__in`, `__gt`, `__contains`, and all other lookup suffixes are resolved through the registration system defined here. No direct import is typically needed — lookups are auto-discovered through field class hierarchies. `[llm-analyzed]`

## Gotchas and edge cases
- Registering a lookup on a field *instance* (via `Field._meta.get_field('name').register_lookup(...)`) takes precedence over class-level registrations. Instance-level lookups are invisible from the class and can cause confusing behavior if registered conditionally. `[llm-analyzed]`
- Calling `expression.as_sql()` directly is explicitly wrong — always use `compiler.compile(expression)` so that backend-specific `as_vendorname()` overrides are invoked. `[llm-analyzed]`
- For spatial fields, `__exact` does geometric vertex-by-vertex equality (SameAsLookup), not SQL `=`. This is a silent but significant semantic difference from non-spatial fields. `[llm-analyzed]`
- Bilateral transforms cannot be used with nested QuerySets — this raises `NotImplementedError` immediately at `Lookup` construction time, before any query is executed. `[llm-analyzed]`
- GIS band indices passed in query tuples are 0-based from the user's perspective (GDALRaster convention) but are stored internally as 1-based (PostGIS convention). Mixing these up produces off-by-one errors in multi-band raster queries. `[llm-analyzed]`

## Referenced by
- [Article](../article/_overview.md)
- [Bar](../bar/_overview.md)
- [Base](../base/_overview.md)
- [Conf](../conf/_overview.md)
- [Constraints](../constraints/_overview.md)
- [Contrib](../contrib/_overview.md)
- [Core](../core/_overview.md)
- [Custom-permissions](../custom-permissions/_overview.md)
- [Custom-user](../custom-user/_overview.md)
- [Customers](../customers/_overview.md)
- [Data](../data/_overview.md)
- [Db](../db/_overview.md)
- [Default-related-name](../default-related-name/_overview.md)
- [Django](../django/_overview.md)
- [Django-views](../django-views/_overview.md)
- [Empty-join](../empty-join/_overview.md)
- [Foo](../foo/_overview.md)
- [Forms](../forms/_overview.md)
- [Invalid-models](../invalid-models/_overview.md)
- [Is-active](../is-active/_overview.md)
- [Minimal](../minimal/_overview.md)
- [Models](../models/_overview.md)
- [Multi-table](../multi-table/_overview.md)
- [Natural](../natural/_overview.md)
- [No-password](../no-password/_overview.md)
- [Person](../person/_overview.md)
- [Proxy](../proxy/_overview.md)
- [Publication](../publication/_overview.md)
- [Query-performing-app](../query-performing-app/_overview.md)
- [Tablespaces](../tablespaces/_overview.md)
- [Templates](../templates/_overview.md)
- [Templatetags](../templatetags/_overview.md)
- [Tenant](../tenant/_overview.md)
- [Tests](../tests/_overview.md)
- [Utils](../utils/_overview.md)
- [Uuid-pk](../uuid-pk/_overview.md)
- [Views](../views/_overview.md)
- [With-custom-email-field](../with-custom-email-field/_overview.md)
- [With-foreign-key](../with-foreign-key/_overview.md)
- [With-integer-username](../with-integer-username/_overview.md)
- [With-many-to-many](../with-many-to-many/_overview.md)
- [With-unique-constraint](../with-unique-constraint/_overview.md)