---
domain: db
confidence: llm-analyzed
last_updated: 2026-04-03
related_files:
  - django/db/transaction.py
  - django/db/backends/postgresql/compiler.py
  - django/db/backends/postgresql/creation.py
  - django/db/backends/postgresql/client.py
  - django/db/backends/postgresql/features.py
  - django/db/backends/postgresql/operations.py
  - django/db/backends/postgresql/introspection.py
  - django/db/backends/postgresql/psycopg_any.py
  - django/db/backends/postgresql/base.py
  - django/db/backends/postgresql/schema.py
  - django/db/backends/signals.py
  - django/db/backends/dummy/features.py
  - django/db/backends/dummy/base.py
  - django/db/backends/oracle/functions.py
  - django/db/backends/oracle/creation.py
  - django/db/backends/oracle/client.py
  - django/db/backends/oracle/features.py
  - django/db/backends/oracle/operations.py
  - django/db/backends/oracle/utils.py
  - django/db/backends/oracle/introspection.py
  - django/db/backends/oracle/base.py
  - django/db/backends/oracle/schema.py
  - django/db/backends/oracle/validation.py
  - django/db/backends/ddl_references.py
  - django/db/backends/utils.py
  - django/db/backends/sqlite3/creation.py
  - django/db/backends/sqlite3/client.py
  - django/db/backends/sqlite3/_functions.py
  - django/db/backends/sqlite3/features.py
  - django/db/backends/sqlite3/operations.py
  - django/db/backends/sqlite3/introspection.py
  - django/db/backends/sqlite3/base.py
  - django/db/backends/sqlite3/schema.py
  - django/db/backends/mysql/compiler.py
  - django/db/backends/mysql/creation.py
  - django/db/backends/mysql/client.py
  - django/db/backends/mysql/features.py
  - django/db/backends/mysql/operations.py
  - django/db/backends/mysql/introspection.py
  - django/db/backends/mysql/base.py
  - django/db/backends/mysql/schema.py
  - django/db/backends/mysql/validation.py
  - django/db/backends/base/creation.py
  - django/db/backends/base/client.py
  - django/db/backends/base/features.py
  - django/db/backends/base/operations.py
  - django/db/backends/base/introspection.py
  - django/db/backends/base/base.py
  - django/db/backends/base/schema.py
  - django/db/backends/base/validation.py
  - django/db/__init__.py
  - django/db/models/options.py
  - django/db/models/signals.py
  - django/db/models/enums.py
  - django/db/models/query.py
  - django/db/models/constants.py
  - django/db/models/expressions.py
  - django/db/models/__init__.py
  - django/db/models/lookups.py
  - django/db/models/indexes.py
  - django/db/models/utils.py
  - django/db/models/aggregates.py
  - django/db/models/deletion.py
  - django/db/models/functions/mixins.py
  - django/db/models/functions/window.py
  - django/db/models/functions/__init__.py
  - django/db/models/functions/comparison.py
  - django/db/models/functions/text.py
  - django/db/models/functions/math.py
  - django/db/models/functions/datetime.py
  - django/db/models/functions/json.py
  - django/db/models/functions/uuid.py
  - django/db/models/fields/related_descriptors.py
  - django/db/models/fields/files.py
  - django/db/models/fields/mixins.py
  - django/db/models/fields/generated.py
  - django/db/models/fields/related.py
  - django/db/models/fields/tuple_lookups.py
  - django/db/models/fields/proxy.py
  - django/db/models/fields/__init__.py
  - django/db/models/fields/composite.py
  - django/db/models/fields/reverse_related.py
  - django/db/models/fields/related_lookups.py
  - django/db/models/fields/json.py
  - django/db/models/constraints.py
  - django/db/models/fetch_modes.py
  - django/db/models/manager.py
  - django/db/models/query_utils.py
  - django/db/models/base.py
  - django/db/models/sql/compiler.py
  - django/db/models/sql/query.py
  - django/db/models/sql/subqueries.py
  - django/db/models/sql/where.py
  - django/db/models/sql/constants.py
  - django/db/models/sql/datastructures.py
  - django/db/utils.py
---

# Db

## What this domain does
The `db` domain is Django's entire database abstraction layer — from raw connection management through the ORM. It is the central hub of the project: nearly every other domain imports from it. Its responsibilities split into three layers: connection infrastructure (connection pooling, routing, error normalization), transaction semantics (atomicity, savepoints, on-commit hooks), and the ORM proper (models, fields, querysets, expressions, migrations). Understanding this domain is prerequisite to working on almost any other domain in the codebase. `[llm-analyzed]`

## Key behaviors
- **Connection management is per-thread and request-scoped**: ConnectionHandler is thread-critical (not just thread-local): each thread gets its own connections, with no cross-thread sharing and no cleanup for async contexts. Connections are reset at `request_started` (queries_log cleared) and closed at both `request_started` and `request_finished` if they are unusable or past their lifetime (CONN_MAX_AGE). The module-level `connection` object is a ConnectionProxy that forwards to `connections['default']` — it exists only for backwards compatibility. New code should use `connections[alias]` directly. `[llm-analyzed]`
- **All backend-specific DB exceptions are normalized to Django's hierarchy**: DatabaseErrorWrapper (a context manager and decorator) catches PEP-249 exceptions from the raw backend and re-raises them as Django's own exception types (DataError, OperationalError, IntegrityError, etc.). Critically, it only sets `errors_occurred = True` on the connection for errors that may make the connection unusable — DataError and IntegrityError do NOT set this flag, because a connection is still valid after a constraint violation or bad data. `[llm-analyzed]`
- **Atomic blocks nest via savepoints; durable blocks explicitly forbid nesting**: The Atomic context manager creates a real transaction at the outermost level and a savepoint for each nested `atomic()` call. If a block is marked `durable=True`, it raises RuntimeError if nested inside any non-test atomic block. The `_from_testcase` flag on atomic blocks allows test harnesses to wrap tests in transactions without triggering the durable guard — this is an intentional testing escape hatch, not a general override. `[llm-analyzed]`
- **on_commit callbacks only fire after the outermost transaction commits**: Functions registered with `on_commit()` are stored on the connection and only invoked when the outermost atomic block exits successfully. If the transaction rolls back, they are discarded. `mark_for_rollback_on_error()` is a low-level utility used by Model.save() to avoid starting a transaction in autocommit mode while still marking the block for rollback if an error occurs within an existing atomic block. `[llm-analyzed]`
- **Field creation order is tracked globally via a counter**: Every Field instance increments a class-level `creation_counter` at instantiation. Django uses this counter to preserve the declaration order of fields on a model. Auto-created fields (like implicit primary keys) use a separate `auto_creation_counter` that starts at -1 and decrements, so they sort before user-defined fields without interfering with the user counter. `[llm-analyzed]`

## Domain interactions
- **Base**: Provides the foundational Model class and ModelBase metaclass that all application models inherit from. The db domain owns all ORM mechanics; base domain models are concrete users of those mechanics. `[llm-analyzed]`
- **Migrations**: The db domain owns the full migration infrastructure (MigrationExecutor, MigrationLoader, MigrationWriter, MigrationAutodetector, ProjectState, etc.). Migration operations (AddField, AlterField, CreateModel, RunPython, RunSQL, etc.) are all defined here and executed against the db connection layer. `[llm-analyzed]`
- **Expressions**: db.models.expressions provides F, Q, Case/When, Subquery, OuterRef, Window, and all expression building blocks. These are consumed by the queryset compiler to build SQL. The expressions domain is logically part of db but separated by file for maintainability. `[llm-analyzed]`
- **Fields**: db.models.fields defines all field types. Fields register their own lookups, define db_type, contribute descriptors to model classes, and drive migration schema detection via non_db_attrs. The fields domain is a direct dependency of db.models.base. `[llm-analyzed]`
- **Conf**: db.utils.ConnectionHandler reads DATABASES from settings to configure connections. The dummy backend fallback when DATABASES is empty or a key is absent is handled entirely within db.utils. `[llm-analyzed]`
- **Dispatch**: db/__init__.py connects reset_queries and close_old_connections to django.core.signals.request_started and request_finished. This is the mechanism by which connections are cleaned up between HTTP requests. `[llm-analyzed]`
- **Core**: Receives signals (request_started, request_finished) from django.core.signals. Also uses django.core.exceptions (ObjectDoesNotExist, ValidationError, FieldError, etc.) throughout the ORM. `[llm-analyzed]`
- **Utils**: Uses django.utils.connection for BaseConnectionHandler and ConnectionProxy base classes. Also uses django.utils.functional (cached_property), django.utils.deprecation, django.utils.module_loading, and various text/encoding utilities. `[llm-analyzed]`

## Gotchas and edge cases
- The module-level `connection` object is a backwards-compatibility proxy to `connections['default']`. It is not thread-safe in multi-database setups — always use `connections[alias]` when working with non-default databases. `[llm-analyzed]`
- DatabaseErrorWrapper does NOT set errors_occurred for IntegrityError or DataError. Only errors that may corrupt the connection state (OperationalError, InternalError, ProgrammingError, etc.) mark the connection as potentially unusable. `[llm-analyzed]`
- Durable atomic blocks check `_from_testcase` to allow test harnesses to wrap tests without triggering the nesting guard. If you set durable=True in production code nested inside a test transaction, the test's _from_testcase flag suppresses the RuntimeError — meaning your code won't be tested in the exact condition it enforces. `[llm-analyzed]`
- TextChoices._generate_next_value_ returns the member name (not a numeric auto-value), so `auto()` in a TextChoices enum produces the string name of the member, not an integer. `[llm-analyzed]`
- Field.non_db_attrs is checked during migration diffing. If you add a custom field attribute that affects the column but is not in non_db_attrs, Django will NOT generate a migration for it. Conversely, adding a purely metadata attribute will generate spurious migrations unless you add it to non_db_attrs. `[llm-analyzed]`

## Notes from code
- [NOTE] There is no need to remap parent dependencies as we can
- [TODO] investigate if old relational fields must be reloaded or if
- [TODO] colorize this SQL code with style.SQL_KEYWORD(), etc.
- [NOTE] We can't modify a dictionary's contents while looping
- [FIXME] Rename possibly_multivalued to multivalued and fix detection
- [TODO] Handle multiple backends with different feature flags.
- [TODO] Handle multiple backends with different feature flags.
- [TODO] It might be possible to trim more joins from the start of

## Dependencies
- [Base](../base/_overview.md)
- [Conf](../conf/_overview.md)
- [Constraints](../constraints/_overview.md)
- [Contrib](../contrib/_overview.md)
- [Core](../core/_overview.md)
- [Dispatch](../dispatch/_overview.md)
- [Django](../django/_overview.md)
- [Django-apps](../django-apps/_overview.md)
- [Django-urls](../django-urls/_overview.md)
- [Django-views](../django-views/_overview.md)
- [Expressions](../expressions/_overview.md)
- [Fields](../fields/_overview.md)
- [Forms](../forms/_overview.md)
- [Http](../http/_overview.md)
- [Indexes](../indexes/_overview.md)
- [Lookups](../lookups/_overview.md)
- [Middleware](../middleware/_overview.md)
- [Options](../options/_overview.md)
- [Proxy](../proxy/_overview.md)
- [Tasks](../tasks/_overview.md)
- [Templates](../templates/_overview.md)
- [Templatetags](../templatetags/_overview.md)
- [Utils](../utils/_overview.md)

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
- [Tasks](../tasks/_overview.md)
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