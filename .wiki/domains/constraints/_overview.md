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
Django's constraints domain provides the building blocks for defining database-level integrity rules that go beyond what field-level validation offers. It solves two distinct problems simultaneously: enforcing data integrity at the database layer (via SQL constraints) and providing Python-level validation (via `validate()`) so application code can catch violations before hitting the DB. The two constraint types — `CheckConstraint` and `UniqueConstraint` — map directly to `CHECK` and `UNIQUE` SQL constructs, but with added integration into Django's validation pipeline. `[llm-analyzed]`

## Key behaviors
- **Dual-layer validation: DB constraint + Python validate()**: Every constraint must implement both SQL generation methods (constraint_sql, create_sql, remove_sql) AND a validate() method. The validate() method issues a live DB query to check the constraint before saving, which means constraint violations can be raised as ValidationError during full_clean() rather than as IntegrityError at save() time. This is critical for user-facing form validation: a ValidationError is catchable and displayable; an IntegrityError from the DB is not. `[llm-analyzed]`
- **non_db_attrs excludes violation_error_* from migration comparison**: The violation_error_code and violation_error_message attributes are listed in non_db_attrs. This means changes to these fields do NOT trigger a new migration — they affect Python-side validation messages only, not the database schema. Developers who change only the error message won't accidentally generate no-op migrations. `[llm-analyzed]`
- **Abstract base class constraint naming uses %(app_label)s/%(class)s interpolation**: Constraint names must be globally unique. On abstract models this is impossible with static names since the same name would be inherited by multiple concrete subclasses. Django solves this by allowing %(app_label)s and %(class)s tokens in the name, which are resolved to the lowercased app label and model class name of the concrete model at migration time. Forgetting this on abstract models causes name collision errors. `[llm-analyzed]`
- **CheckConstraint rejects non-conditional expressions at __init__ time**: The CheckConstraint constructor checks `condition.conditional` and raises TypeError immediately if the condition is not a Q object or boolean expression. This fails fast rather than producing a cryptic database error later during migrations. `[llm-analyzed]`
- **CheckConstraint with RawSQL emits W045 warning and skips Python validation**: If a CheckConstraint's condition contains a RawSQL() expression, Django cannot validate it at the Python level (validate() is skipped for that constraint). This is surfaced as a system check warning W045, not a hard error. The DB constraint is still created, but full_clean() will not catch violations for it. `[llm-analyzed]`

## Domain interactions
- **Expressions**: Uses ExpressionList, F, Exists, and RawSQL to build constraint SQL. The contains_expressions property on BaseConstraint signals whether the constraint uses arbitrary expressions (as opposed to simple field references), which affects how schema editors handle them. `[llm-analyzed]`
- **Indexes**: Imports IndexExpression to build expression-based unique constraints, sharing the same SQL generation machinery used for functional indexes. UniqueConstraint.expressions follows the same database restrictions as Index.expressions. `[llm-analyzed]`
- **Lookups**: Uses Exact and IsNull lookups internally to build the SQL queries that back Python-level validate() calls on UniqueConstraint and CheckConstraint. `[llm-analyzed]`
- **Options**: Constraints are registered on models via Meta.constraints. The options domain is responsible for collecting and exposing constraints from the model's Meta class. Constraint names must be unique across the entire constraints list. `[llm-analyzed]`
- **Core (checks)**: Emits system check errors and warnings (E041, W027, W045) through django.core.checks. These are surfaced during manage.py check and prevent invalid constraint definitions from reaching the database. `[llm-analyzed]`
- **Db**: Constraint SQL is generated through schema_editor (passed in from the db layer). The constraint's deferrable setting and feature detection (supports_table_check_constraints, supports_deferrable_unique_constraints) are checked against the active database connection's features. `[llm-analyzed]`
- **Utils (translation)**: The default_violation_error_message is wrapped with gettext_lazy for i18n. Custom violation_error_message values passed by the developer are NOT automatically translated — they must be wrapped with _() by the caller if translation is needed. `[llm-analyzed]`

## Gotchas and edge cases
- Changing violation_error_message or violation_error_code never generates a migration — those fields are in non_db_attrs. If you need DB changes, you must change the constraint itself. `[llm-analyzed]`
- RawSQL in a CheckConstraint condition silently disables Python-level validation (validate() is skipped). The DB constraint still exists, but full_clean() won't catch violations. `[llm-analyzed]`
- On Oracle < 23c, CHECK constraints on nullable fields must explicitly allow NULL values. Q(age__gte=18) alone will cause divergence between Python validate() and DB enforcement. `[llm-analyzed]`
- Constraints on abstract models MUST use %(app_label)s_%(class)s in the name. A static name will cause name collision errors when multiple concrete models inherit the abstract base. `[llm-analyzed]`
- FK traversal (joined field references) in constraint conditions is rejected at system check time with E041. Constraints can only reference columns on the same table. `[llm-analyzed]`

## Dependencies
- [Base](../base/_overview.md)
- [Core](../core/_overview.md)
- [Db](../db/_overview.md)
- [Django](../django/_overview.md)
- [Expressions](../expressions/_overview.md)
- [Indexes](../indexes/_overview.md)
- [Lookups](../lookups/_overview.md)
- [Options](../options/_overview.md)
- [Utils](../utils/_overview.md)

## Referenced by
- [Article](../article/_overview.md)
- [Bar](../bar/_overview.md)
- [Base](../base/_overview.md)
- [Conf](../conf/_overview.md)
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