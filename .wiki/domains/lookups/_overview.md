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
The lookups domain is Django's mechanism for translating Python filter expressions (e.g., `field__exact=value`, `field__gt=5`) into database-specific SQL WHERE clauses. It sits at the boundary between Python ORM semantics and raw SQL, handling type coercion, operator translation, database-specific quirks, and value validation. Every `QuerySet.filter()` call routes through this domain. It is foundational infrastructure: almost every other domain in this codebase depends on it to express queries. `[llm-analyzed]`

## Key behaviors
- **Two-class hierarchy: Lookup vs BuiltinLookup**: The base `Lookup` class handles generic expression lookups with no assumption about operator syntax. `BuiltinLookup` extends it with two key additions: (1) it applies `connection.ops.lookup_cast()` to the LHS to type-cast it in SQL when the backend requires it, and (2) it retrieves the actual SQL operator from `connection.operators[self.lookup_name]` via `get_rhs_op()`. This means all Django built-in lookups (`exact`, `gt`, `contains`, etc.) are BuiltinLookups, while custom or GIS lookups can extend the base `Lookup` directly with their own `as_sql()` implementation. `[llm-analyzed]`
- **Value preparation is separated from SQL generation via FieldGetDbPrepValue mixins**: `FieldGetDbPrepValueMixin` (for single values) and `FieldGetDbPrepValueIterableMixin` (for lists/ranges) exist because field types need type-aware value preparation that differs from the generic `get_db_prep_lookup()` fallback. They call `field.get_db_prep_value()` on each value, which handles things like converting Python objects to database-compatible types (e.g., Python `datetime` to DB timestamp string, FK Python object to integer ID). Without these mixins, all values would pass through as raw Python objects with just `('%s', (value,))` formatting. `[llm-analyzed]`
- **Bilateral transforms apply to both sides of a comparison**: When a `Transform` is marked `bilateral = True`, it is applied to both the LHS (field) and every value in the RHS before the comparison is made. This is collected during `Lookup.__init__()` via `lhs.get_bilateral_transforms()`. The classic example is `Unaccent`: `name__unaccent__contains='cafe'` will apply unaccent to both the stored field value and the search term, so accented characters are transparently matched. Bilateral transforms cannot be used with nested QuerySet RHS — this is explicitly checked and raises `NotImplementedError`. `[llm-analyzed]`
- **EmptyResultSet and FullResultSet short-circuit SQL generation**: Several lookups raise `EmptyResultSet` (to produce a WHERE clause equivalent to FALSE) or `FullResultSet` (equivalent to TRUE) instead of generating real SQL. This avoids unnecessary database round-trips. Key cases: `In` with an empty list or a list of only `None` values raises `EmptyResultSet`. `IsNull` on a constant `Value(None)` raises `FullResultSet` (if checking IS NULL) or `EmptyResultSet` (if checking IS NOT NULL). Integer overflow lookups use this pattern too: filtering an IntegerField with a value outside the field's type range raises `EmptyResultSet` or `FullResultSet` depending on the operator direction. `[llm-analyzed]`
- **In lookup removes None values silently and handles max IN list sizes**: SQL `NULL IN (...)` is never true, so `In.process_rhs()` silently discards `None` from the RHS list using `OrderedSet.discard(None)`. If removing `None` empties the list entirely, `EmptyResultSet` is raised. Additionally, some databases (notably Oracle) limit how many elements can appear in an IN clause. The lookup detects this via `connection.ops.max_in_list_size()` and splits a large list into multiple OR-connected IN clauses. `[llm-analyzed]`

## Domain interactions
- **Expressions**: Lookup extends Expression directly, making every lookup usable as a query expression (in annotations, Case/When, etc.). It uses Expression infrastructure for resolve_expression(), get_source_expressions(), and get_group_by_cols(). The rhs can be any Expression subclass, not just raw values — this is how subquery lookups work. `[llm-analyzed]`
- **Fields**: Fields are the primary registration target for lookups. Field.get_lookup() is the resolution entry point for filter parsing. Field.get_db_prep_value() is called by FieldGetDbPrepValueMixin to type-convert RHS values. Fields also define output_field on their expressions, which lookups use to determine value preparation behavior. `[llm-analyzed]`
- **Query_utils**: RegisterLookupMixin (from query_utils) provides the entire registration API. Lookup classes use @Field.register_lookup as a class decorator to self-register. The MRO-based lookup resolution in RegisterLookupMixin is what makes subclass lookup inheritance work transparently. `[llm-analyzed]`
- **Django (ORM compiler/backend)**: Lookups are consumed by the SQL compiler. compiler.compile(lookup) dispatches to as_sql() or as_vendorname(). connection.operators[], connection.ops.lookup_cast(), connection.ops.integer_field_range(), and connection.ops.max_in_list_size() are all called from within lookup implementations to adapt SQL to the active backend. `[llm-analyzed]`
- **Gis**: GISLookup and RasterBandTransform extend core Lookup/Transform. The GIS domain adds spatial operator dispatch, raster band handling, and geometry adapter wrapping. It relies on connection.ops.gis_operators[] and connection.ops.Adapter() from the GIS database backend operations. `[llm-analyzed]`

## Gotchas and edge cases
- None in an In lookup list is silently dropped, not treated as NULL IS NULL. There is no way to include a NULL match via the In lookup — use Q(field__isnull=True) | Q(field__in=[...]) instead. `[llm-analyzed]`
- Bilateral transforms cannot be used with subquery (QuerySet) RHS values. The NotImplementedError is raised in Lookup.__init__, not at query execution time. `[llm-analyzed]`
- PostgresOperatorLookup lookups (HasKey, Overlap, etc.) have no as_sql() fallback. Using them on non-PostgreSQL backends will either raise NotImplementedError or produce invalid SQL silently. `[llm-analyzed]`
- UUIDTextMixin modifies self.rhs in-place by replacing it with a SQL Replace() expression during process_rhs(). This means the lookup object is mutated on first compilation — do not compile the same lookup instance twice on different connections. `[llm-analyzed]`
- IntegerField float rounding uses math.ceil, not round(). Filtering with field__lt=2.5 will be interpreted as field__lt=3, not field__lt=2. This is intentional but can be surprising. `[llm-analyzed]`

## Referenced by
- [Article](../article/_overview.md)
- [Bar](../bar/_overview.md)
- [Base](../base/_overview.md)
- [Custom-permissions](../custom-permissions/_overview.md)
- [Custom-user](../custom-user/_overview.md)
- [Customers](../customers/_overview.md)
- [Data](../data/_overview.md)
- [Default-related-name](../default-related-name/_overview.md)
- [Django](../django/_overview.md)
- [Empty-join](../empty-join/_overview.md)
- [Expressions](../expressions/_overview.md)
- [Fields](../fields/_overview.md)
- [Foo](../foo/_overview.md)
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
- [Tenant](../tenant/_overview.md)
- [Tests](../tests/_overview.md)
- [Uuid-pk](../uuid-pk/_overview.md)
- [Views](../views/_overview.md)
- [With-custom-email-field](../with-custom-email-field/_overview.md)
- [With-foreign-key](../with-foreign-key/_overview.md)
- [With-integer-username](../with-integer-username/_overview.md)
- [With-many-to-many](../with-many-to-many/_overview.md)
- [With-unique-constraint](../with-unique-constraint/_overview.md)