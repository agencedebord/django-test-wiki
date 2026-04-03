---
domain: expressions
confidence: llm-analyzed
last_updated: 2026-04-03
related_files:
  - docs/ref/models/expressions.txt
  - django/db/models/expressions.py
---

# Expressions

## What this domain does
The expressions domain is Django's Python DSL for describing database-side computations. It allows callers to build SQL expressions — arithmetic, conditionals, window functions, subqueries, and more — as composable Python objects that are compiled to SQL at query execution time. The core contract is: build an expression tree in Python → resolve it against a specific query (binding field types and join paths) → compile it to a `(sql, params)` tuple. No data ever leaves the database just because an expression is constructed. `[llm-analyzed]`

## Key behaviors
- **Two-phase lifecycle: resolve then compile**: Every expression has a `resolve_expression()` step that binds the expression to a concrete query (resolving field names, validating join permissions, propagating `is_summary`), followed by `as_sql()` which emits the actual SQL. `resolve_expression()` always returns a copy — the original expression object is never mutated. This means the same expression instance can safely be used in multiple queries. `[llm-analyzed]`
- **Output field type inference is intentionally fallible**: `BaseExpression._resolve_output_field()` infers the output type by checking that all source fields match. The code comment explicitly calls this 'mostly a bad idea' but it's preserved for backward-compat with third-party `Func` subclasses. Mixed types raise `FieldError` and require an explicit `output_field=`. The `_connector_combinations` table is the authoritative registry for valid cross-type arithmetic (e.g. IntegerField + FloatField → FloatField). Missing combinations raise `FieldError` by design. `[llm-analyzed]`
- **CombinedExpression auto-promotes to DurationExpression or TemporalSubtraction**: During `resolve_expression()`, if one operand is a `DurationField` (and the other isn't), the `CombinedExpression` transparently upgrades to a `DurationExpression`, which knows how to call `connection.ops.format_for_duration_arithmetic()` and `combine_duration_expression()` — these differ across backends. Similarly, subtracting two temporal fields (DateField - DateField, DateTimeField - DateTimeField, etc.) auto-promotes to `TemporalSubtraction`, which delegates to `connection.ops.subtract_temporals()`. This promotion happens post-resolution, so the static type of the expression changes. `[llm-analyzed]`
- **F is not a BaseExpression — it is only Combinable**: `F` inherits from `Combinable` directly, not from `BaseExpression`. It has no `output_field` until resolved via `query.resolve_ref()`. This means F instances cannot be compiled to SQL directly; they must first be resolved against a query. Attempting to access `output_field` on an unresolved F will raise an `AttributeError`, not `OutputFieldIsNoneError`. `[llm-analyzed]`
- **OuterRef resolves to ResolvedOuterRef, which errors if used as top-level SQL**: `OuterRef('field')` resolves to a `ResolvedOuterRef`, which overrides `as_sql()` to always raise `ValueError`. This enforces that outer references are only valid inside a correlated subquery. `ResolvedOuterRef.resolve_expression()` also sets `col.possibly_multivalued = LOOKUP_SEP in self.name`, marking traversals through relationships as potentially multi-valued. The FIXME in the code notes this detection is imprecise — it flags any dotted path, not just actual M2M/O2M joins. `[llm-analyzed]`

## Domain interactions
- **Fields**: Expressions delegate output type resolution, DB value preparation (get_db_prep_value/get_db_prep_save), value conversion (get_db_converters), and SQL placeholder generation (get_placeholder_sql) to field instances. The expression layer treats fields as its type system and serialization layer. `[llm-analyzed]`
- **Lookups**: Expressions expose get_lookup() and get_transform() by forwarding to output_field. This allows expressions to participate in the double-underscore lookup chain (e.g. annotated values can be further filtered with __gt, __icontains, etc.). `[llm-analyzed]`
- **Constraints**: Expressions expose `constraint_validation_compatible` and `get_expression_for_validation()`. Expressions that cannot be used during constraint validation (e.g. Window) return their inner expression instead of themselves, so constraint checking can still validate the underlying column value. `[llm-analyzed]`
- **Base (query compiler)**: Expressions receive a `compiler` object in `as_sql()` that has a `compile(expr)` method. The compiler drives SQL generation by recursively compiling source expressions. Expressions also receive a `connection` object to check backend features and delegate vendor-specific SQL (e.g. combine_expression, subtract_temporals). `[llm-analyzed]`
- **Options**: RawSQL.resolve_expression() inspects query.model._meta.all_parents to detect if raw SQL references parent model columns, ensuring parent table joins are added when needed for multi-table inheritance. `[llm-analyzed]`
- **Django (database backends)**: Expressions call connection.ops methods for backend-specific behavior: combine_expression, combine_duration_expression, subtract_temporals, format_for_duration_arithmetic, window_frame_rows_start_end, window_frame_range_start_end, unification_cast_sql, and check_expression_support. The vendor dispatch mechanism (as_sqlite, as_mysql, as_oracle) allows expressions to override SQL generation per backend by defining as_{vendor} methods. `[llm-analyzed]`

## Gotchas and edge cases
- F is not a BaseExpression — it cannot be directly compiled to SQL. Any code path that calls as_sql() on an unresolved F will get AttributeError, not a useful error message. `[llm-analyzed]`
- The `conditional` property is `isinstance(self.output_field, BooleanField)`. If output_field hasn't been resolved yet (raises OutputFieldIsNoneError), accessing `conditional` will propagate that error, which can be surprising in operator overload chains. `[llm-analyzed]`
- Integer division (`/`) in CombinedExpression follows Postgres/SQLite behavior (floor division for integer operands), not MySQL/Oracle behavior. This is documented in a code comment but is a silent behavioral difference across backends. `[llm-analyzed]`
- cached_property values (contains_aggregate, contains_over_clause, etc.) are set on the instance on first access and never invalidated. If you mutate source expressions after first access, these properties will return stale values. `[llm-analyzed]`
- `OuterRef` used inside a nested subquery of depth > 1 requires wrapping in another `OuterRef`: `OuterRef(OuterRef('field'))`. A plain `OuterRef('field')` only reaches one level up. `[llm-analyzed]`

## Notes from code
- [FIXME] Rename possibly_multivalued to multivalued and fix detection

## Dependencies
- [Base](../base/_overview.md)
- [Constraints](../constraints/_overview.md)
- [Django](../django/_overview.md)
- [Indexes](../indexes/_overview.md)
- [Lookups](../lookups/_overview.md)
- [Options](../options/_overview.md)

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
- [Fields](../fields/_overview.md)
- [Foo](../foo/_overview.md)
- [Indexes](../indexes/_overview.md)
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