---
domain: constraints
confidence: llm-analyzed
last_updated: 2026-04-03
related_files:
  - docs/ref/models/constraints.txt
  - django/db/models/constraints.py
---

# Constraints

## What this domain does
The constraints domain provides Django's mechanism for declaring database-level integrity rules that are enforced both at the database layer (via DDL) and at the application layer (via model validation). It bridges the gap between Python-declared rules and database-specific SQL, allowing developers to write constraints once and have them work across supported databases. `[llm-analyzed]`

## Key behaviors
- **Dual enforcement: DB and application validation**: Each constraint is enforced at two levels: (1) as actual database DDL constraints (CHECK, UNIQUE) via constraint_sql/create_sql/remove_sql, and (2) at Python validation time via the validate() method, which performs a database query to simulate what the DB would reject. This means constraints are checked during model.full_clean() before a save, not only when the DB rejects an INSERT/UPDATE. The validate() method intentionally skips validation if any field in the exclude list is needed — it silently passes rather than raising a false positive. `[llm-analyzed]`
- **non_db_attrs controls migration diffing**: The non_db_attrs tuple on BaseConstraint lists attributes that should NOT trigger a migration when changed. violation_error_code and violation_error_message are in this list because they only affect Python-side validation error messages, not the actual database constraint. Subclasses can extend this tuple to exclude additional attrs from migration detection. `[llm-analyzed]`
- **Abstract base class name interpolation**: Constraint names must be globally unique. To allow constraints on abstract models, the name can contain %(app_label)s and %(class)s placeholders, which Django replaces with the concrete model's lowercased app label and class name. Without this, inheriting an abstract model with a constraint would produce name collisions across all subclasses. `[llm-analyzed]`
- **CheckConstraint validates condition type at init**: CheckConstraint.__init__ immediately checks that the condition has a conditional attribute (i.e., it's a Q object or a boolean Expression). It raises TypeError at construction time rather than failing silently or at migration time. This is an eager validation pattern to catch misconfiguration before deployment. `[llm-analyzed]`
- **RawSQL in CheckConstraint disables Python-side validation**: If a CheckConstraint's condition contains any RawSQL expression, Django emits a W045 warning during system checks and skips the validate() call for that constraint. This is because RawSQL is opaque to the ORM — Django cannot safely re-execute it as a Python-side lookup. The constraint still exists at the DB level; only the model.full_clean() check is skipped. `[llm-analyzed]`

## Domain interactions
- **Django.db.models (Query, Q, F, Exists, ExpressionList)**: CheckConstraint uses Query.build_where() to convert a Q object into SQL WHERE clause SQL for the database CHECK constraint DDL. UniqueConstraint uses ExpressionList and IndexExpression (from the indexes module) to serialize functional expressions. The constraint domain is a consumer of the ORM's expression/query building infrastructure — it does not extend it. `[llm-analyzed]`
- **Django.core.checks**: Both CheckConstraint and UniqueConstraint implement check(model, connection) to emit system check errors/warnings. These are registered via Model._check_constraints and run during ./manage.py check. The constraint domain uses checks.Error (E041, models.E348) and checks.Warning (W027, W045) to surface configuration problems before runtime. `[llm-analyzed]`
- **Schema editor (django.db.backends.*.schema)**: constraint_sql(), create_sql(), and remove_sql() receive a schema_editor instance and use its connection and quote_value() method to produce database-specific SQL. The constraint domain does not know which database is being used — that is abstracted by the schema editor passed in at migration time. `[llm-analyzed]`
- **Meta.constraints option (consuming models)**: Constraints are declared in a model's Meta.constraints list. The model infrastructure calls constraint.check() during system checks, constraint.create_sql() during migrations, and constraint.validate() during model.full_clean(). The constraints domain provides this interface; the model domain is responsible for invoking it at the right lifecycle points. `[llm-analyzed]`

## Gotchas and edge cases
- Constraint names must be unique across the entire database, not just per-model. Django does not enforce global uniqueness at definition time — conflicts only surface when running migrations against a real DB. `[llm-analyzed]`
- The deconstruct() path rewrites django.db.models.constraints to django.db.models for cleaner migration files. Custom constraint subclasses that live in other modules will NOT get this path rewriting — their full module path will appear in migrations. `[llm-analyzed]`
- violation_error_message defaults to the class-level default_violation_error_message, not None. If you check `instance.violation_error_message is None`, it will always be False after __init__ runs — the attribute is always set to either the custom value or the default. `[llm-analyzed]`
- UniqueConstraint with a condition creates a partial unique index, not a CHECK constraint. This means the uniqueness is only enforced for rows matching the condition — rows not matching the condition are completely ignored by the constraint. `[llm-analyzed]`
- UniqueConstraint.nulls_distinct=False (PostgreSQL 15+) makes NULL values count as equal for uniqueness purposes. The default (None) uses database-default behavior, which traditionally treats NULLs as distinct. This parameter has no effect on databases that don't support it. `[llm-analyzed]`

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