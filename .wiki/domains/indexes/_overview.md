---
domain: indexes
confidence: llm-analyzed
last_updated: 2026-04-03
related_files:
  - docs/ref/models/indexes.txt
  - django/db/models/indexes.py
---

# Indexes

## What this domain does
The `indexes` domain provides Django's `Index` class — the mechanism for declaring database indexes on models via `Meta.indexes`. It handles the full lifecycle of index definition, validation, and SQL generation, abstracting over significant cross-database differences in index feature support. `[llm-analyzed]`

## Key behaviors
- **Mutual exclusivity between fields and expressions**: An Index must use either `fields` (column names) or `*expressions` (functional index expressions) — never both. This is enforced at construction time with a hard ValueError. The two modes result in different SQL generation paths: field-based indexes use column references while expression-based indexes go through IndexExpression wrapping with set_wrapper_classes() to adapt the expression tree for the specific database backend. `[llm-analyzed]`
- **Name is required for advanced features**: A name is mandatory when using: `opclasses`, `condition` (partial index), `*expressions` (functional index), or `include` (covering index). Auto-generated names are only available for simple field-based indexes without any of these features. This is enforced at construction with explicit ValueError messages for each case. `[llm-analyzed]`
- **Field ordering via hyphen prefix**: Descending order for a field is expressed by prefixing the field name with a hyphen (e.g., `'-pub_date'`). The constructor strips this prefix and stores the result in `fields_orders` as a list of (field_name, order_direction) tuples, where order_direction is either '' (ascending) or 'DESC'. This parsing happens once at construction; the hyphen syntax is a Django convention not present at the database level. `[llm-analyzed]`
- **Covering indexes via include (non-key columns)**: The `include` parameter adds non-key columns to the index for index-only scan support. These columns cannot be used for filtering or sorting — they only allow queries to be satisfied entirely from the index without touching the heap. The feature is silently ignored with a Warning (W040) on databases that don't support it (non-PostgreSQL databases primarily). `[llm-analyzed]`
- **Partial indexes via condition, with broad database incompatibility**: `condition` accepts a Q object and generates a WHERE clause in the index definition. MySQL, MariaDB, and Oracle do not support partial indexes. MySQL/MariaDB silently ignore the condition. Oracle requires a workaround using functional indexes with Case expressions. SQLite supports partial indexes with some restrictions. The check() method emits Warning W037 on unsupported databases rather than raising an error. `[llm-analyzed]`

## Domain interactions
- **Db/backends**: Index.create_sql() delegates to schema_editor for actual SQL emission. It relies on connection.features flags (supports_partial_indexes, supports_covering_indexes, supports_expression_indexes) to gate behavior and issue warnings. The IndexExpression.set_wrapper_classes() call uses the connection to adapt expression trees for backend-specific SQL generation. `[llm-analyzed]`
- **Db/models/expressions**: Functional indexes use ExpressionList, F, Func, OrderBy, and Col from the expressions module. The Index class wraps raw expressions in IndexExpression (a subclass of Func) before compiling them, ensuring they emit valid index-specific SQL rather than query-context SQL. `[llm-analyzed]`
- **Db/models/query_utils (Q)**: The condition parameter must be a Q instance. The Q object is compiled to SQL via a temporary Query object in _get_condition_sql(), effectively reusing the ORM's WHERE clause compiler to produce the partial index predicate. `[llm-analyzed]`
- **Contrib/postgres/indexes**: The base Index class covers B-Tree indexes. PostgreSQL-specific index types (GIN, GiST, BRIN, Hash, SpGiST) are implemented as subclasses in django.contrib.postgres.indexes. The opclasses-with-expressions limitation is explicitly addressed there via the OpClass() expression wrapper. `[llm-analyzed]`
- **Core/checks**: The check() method integrates with Django's system check framework, emitting errors E033/E034 for invalid names and warnings W037/W040/W043 for features unsupported by the current database connection. These are surfaced during manage.py check and before migrations. `[llm-analyzed]`

## Gotchas and edge cases
- PostgreSQL requires all functions in expression indexes and partial index conditions to be marked IMMUTABLE. Django does not validate this — PostgreSQL will error at migration time. Functions like Concat() and date functions will silently appear valid in Python but fail when creating the migration. `[llm-analyzed]`
- Oracle requires functions in expression indexes to be marked DETERMINISTIC. Same issue: Django doesn't validate, Oracle errors at migration time. Random() and similar functions will fail. `[llm-analyzed]`
- MariaDB silently ignores both functional indexes and partial index conditions without any warning or error. An index defined with expressions on MariaDB creates a regular index or nothing, with no indication that the expression was dropped. `[llm-analyzed]`
- The `opclasses` and expression-based index (`*expressions`) arguments are mutually exclusive at the Python level. To use operator classes with expressions on PostgreSQL, you must use `OpClass()` from `django.contrib.postgres.indexes` wrapped around the expression inside the `*expressions` arguments. `[llm-analyzed]`
- Auto-generated index names are non-deterministic across databases and may exceed backend limits if field names are long. Always specify `name` explicitly for any index with a condition, expression, opclasses, or include to avoid silent name truncation or errors. `[llm-analyzed]`

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