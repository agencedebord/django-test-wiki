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
The indexes domain provides a database-agnostic abstraction for defining indexes on Django models. Its core job is to translate Python-level index specifications (field names, expressions, conditions, covering columns) into database DDL statements at migration time, while validating constraints at model-check time. It bridges the gap between Django's ORM expression layer and the schema editor layer that generates actual SQL. `[llm-analyzed]`

## Key behaviors
- **Mutual exclusion between fields and expressions**: An index can be defined either via field names (strings) or ORM expressions, never both simultaneously. This is enforced in __init__ with a ValueError. When expressions are used, string arguments are automatically promoted to F() objects so callers can mix bare strings and expression objects in *expressions without wrapping. `[llm-analyzed]`
- **Name is mandatory for non-trivial indexes**: Using opclasses, condition, include, or *expressions all require an explicit name. Only a plain field-based index can have its name auto-generated. This is because these features produce indexes that cannot be safely renamed by Django's auto-naming (which uses a hash of field names) and need a stable identity across migrations. `[llm-analyzed]`
- **Auto-generated name derivation**: When no name is provided, Django generates one using a digest of the model and field names, suffixed with 'idx'. The max length is hard-capped at 30 characters for Oracle cross-database compatibility. Auto-generated names also cannot start with a digit or underscore — these constraints are enforced both at construction (via check()) and are documented as Oracle restrictions. `[llm-analyzed]`
- **Field ordering via hyphen prefix convention**: Descending order on a field is specified by prefixing the field name with '-' (e.g., '-pub_date'). The __init__ strips the hyphen and stores a parallel list of (field_name, order_direction) tuples in fields_orders. This mirrors Django's queryset ordering convention intentionally. `[llm-analyzed]`
- **Condition SQL compilation is deferred to schema time**: _get_condition_sql() builds a Query object and compiles the Q condition through the query compiler at schema-editor time, not at model definition time. Parameters are eagerly quoted using schema_editor.quote_value() rather than passed as bind parameters, because index DDL is not a parameterized query. `[llm-analyzed]`

## Domain interactions
- **Expressions**: Imports Col, ExpressionList, F, Func, OrderBy from expressions. ExpressionList is used to group IndexExpression wrappers into a single resolvable unit. F() is used to promote bare string expression arguments. The entire expression resolution pipeline (resolve_expression against a Query) is reused from expressions. `[llm-analyzed]`
- **Base (django.db.backends.utils)**: Uses names_digest() and split_identifier() for auto-generating index names. These utilities ensure names are hash-stable and properly split for schema operations. `[llm-analyzed]`
- **Django (schema editor)**: create_sql() is called by the schema editor during migration. It receives schema_editor as a parameter to access connection features, quote_value(), and the connection itself. The index does not execute DDL — it only generates the SQL string that the schema editor executes. `[llm-analyzed]`
- **Django (model _meta and checks)**: check() calls model._check_local_fields() to validate that all referenced field names (from fields, include, and expression references) actually exist on the model. It also reads model._meta.indexes to emit database-feature warnings for all indexes on the model at once, not just the current one. `[llm-analyzed]`

## Gotchas and edge cases
- The database-feature warning check in check() iterates over all indexes on the model (model._meta.indexes) to detect if any index uses a feature the DB doesn't support — but it only emits one warning per feature, regardless of how many indexes violate it. This means the warning is model-level, not index-level. `[llm-analyzed]`
- opclasses cannot be combined with *expressions — for expressions that need operator class control on PostgreSQL, callers must use django.contrib.postgres.indexes.OpClass() wrapper instead. This is enforced with a ValueError in __init__. `[llm-analyzed]`
- MariaDB silently ignores functional (expression-based) indexes entirely. The W043 warning is only emitted if connection.features.supports_expression_indexes is False, so this is detectable at check time. `[llm-analyzed]`
- Oracle does not support partial indexes (condition). The docs note they can be emulated using functional indexes with Case expressions, but Django provides no automatic fallback — the condition is simply ignored with a Warning. `[llm-analyzed]`
- Abstract base class index names must use %(app_label)s and %(class)s interpolation tokens to avoid name collisions when the abstract class is inherited by multiple concrete models. Django will substitute these at model registration time, but if you forget and use a static name, all subclasses share the same index name, causing migration conflicts. `[llm-analyzed]`

## Dependencies
- [Base](../base/_overview.md)
- [Django](../django/_overview.md)
- [Expressions](../expressions/_overview.md)

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