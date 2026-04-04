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
The `expressions` domain is Django's core abstraction for describing SQL computations in Python. It lets you build database-level operations (arithmetic, conditionals, subqueries, window functions) as composable Python objects that get compiled to SQL at query time — without pulling data into Python memory first. Almost every other domain in this project depends on it because it underpins F(), annotations, aggregations, ordering, filtering, and default values. `[llm-analyzed]`

## Key behaviors
- **F() defers field reads to the database**: F('field') does not read the field value into Python — it generates SQL that references the column. This means `obj.field = F('field') + 1; obj.save()` becomes a single atomic UPDATE, avoiding a race condition that would exist if you read-increment-write in Python. This is intentional: Python never sees the value. `[llm-analyzed]`
- **Combinable wraps non-expression operands automatically**: When you combine an expression with a plain Python value (e.g. F('price') * 2), _combine() wraps the non-expression side in Value() automatically. This means any Python literal used in arithmetic with an expression is silently promoted to a Value node. `[llm-analyzed]`
- **& | ^ are overloaded with conditional-aware branching**: On Combinable, __and__, __or__, __xor__ check whether both operands have conditional=True. If so, they delegate to Q() boolean logic. If not, they raise NotImplementedError and tell the caller to use .bitand()/.bitor()/.bitxor() instead. This prevents silently mixing boolean and bitwise semantics. `[llm-analyzed]`
- **SQLiteNumericMixin patches DecimalField output for SQLite**: SQLite does not have a native DECIMAL type. Any expression whose output_field is a DecimalField gets its SQL wrapped in CAST(... AS NUMERIC) when compiled for SQLite. This mixin is mixed into expressions that need it; if output_field cannot be determined (FieldError), the cast is silently skipped. `[llm-analyzed]`
- **BaseExpression carries several capability flags**: BaseExpression defines boolean class-level flags: filterable (can appear in WHERE), window_compatible (can be a Window source expression), allowed_default (can be used as a database-level column default), constraint_validation_compatible, and set_returning (may return multiple rows, like a subquery). These flags are checked by the ORM compiler before emitting SQL — subclasses must override them if they deviate from the defaults. `[llm-analyzed]`

## Domain interactions
- **Db / compiler**: Expressions are compiled to SQL strings by the database compiler. Each expression implements as_sql() (and optionally as_sqlite(), as_oracle(), etc.) which the compiler calls. The compiler passes itself and the connection object; expressions use these to emit dialect-specific SQL. `[llm-analyzed]`
- **Fields**: output_field on an expression is always a model field instance (e.g. IntegerField()). The expressions domain imports from django.db.models.fields to resolve types, perform type coercion after database reads, and determine whether CAST wrappers are needed (e.g. DecimalField on SQLite). `[llm-analyzed]`
- **Query_utils (Q)**: Combinable's boolean operators (__and__, __or__, __xor__) delegate to Q() when both operands are conditional expressions. This means the expressions and Q-object systems are interoperable: boolean expressions can be composed with Q combinators transparently. `[llm-analyzed]`
- **All dependent domains (article, bar, base, constraints, etc.)**: This domain is the foundation for every queryset operation that is not a simple field lookup. Every annotation, aggregation, ordering expression, conditional (Case/When), subquery (Subquery/Exists), and window function (Window) is a subclass or consumer of BaseExpression. Dependent domains import F, Value, Case, When, Subquery, Exists, OuterRef, etc. from here. `[llm-analyzed]`

## Gotchas and edge cases
- F() assignments on model instances (obj.field = F('field') + 1) do not refresh the instance after save(). The Python attribute still holds the F() expression object, not the new database value. You must call obj.refresh_from_db() to get the actual value back. `[llm-analyzed]`
- & and | on expressions raise NotImplementedError unless both sides are conditional (boolean) expressions. This is a common footgun: `F('a') & F('b')` fails; you must use .bitand() for bitwise ops. `[llm-analyzed]`
- output_field inference fails silently for some mixed-type expressions — Django raises OutputFieldIsNoneError (a subclass of FieldError) at query compilation time, not when the expression is constructed. The error only surfaces when the queryset is evaluated. `[llm-analyzed]`
- SQLiteNumericMixin's CAST wrapping is suppressed if output_field raises FieldError. This means a malformed expression on SQLite may silently produce wrong numeric results rather than raising an error. `[llm-analyzed]`
- The MOD connector is stored as '%%' (double percent) rather than '%'. This is intentional: the string is later interpolated into a format string, and a single % would be consumed by Python's string formatting, corrupting the SQL. `[llm-analyzed]`

## Notes from code
- [FIXME] Rename possibly_multivalued to multivalued and fix detection

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