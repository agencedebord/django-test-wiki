---
domain: models
confidence: llm-analyzed
last_updated: 2026-04-03
related_files:
  - django/db/models/__init__.py
  - django/db/models/base.py
  - django/db/models/query.py
  - django/db/models/manager.py
  - django/db/models/options.py
  - django/db/models/deletion.py
  - django/db/models/signals.py
  - django/db/models/expressions.py
  - django/db/models/constraints.py
  - django/db/models/enums.py
  - django/db/models/fetch_modes.py
  - django/db/models/aggregates.py
  - django/db/models/lookups.py
  - django/db/models/indexes.py
  - django/db/models/query_utils.py
  - django/db/models/utils.py
  - django/db/models/fields/__init__.py
  - django/db/models/fields/composite.py
  - django/db/models/fields/files.py
  - django/db/models/fields/generated.py
  - django/db/models/fields/json.py
  - django/db/models/fields/related.py
  - django/db/models/fields/related_descriptors.py
  - django/db/models/sql/query.py
  - django/db/models/sql/compiler.py
  - django/db/models/sql/where.py
---

# Models

## What this domain does

This is the core ORM (Object-Relational Mapping) layer of Django. It provides the `Model` base class and the entire machinery for defining database-backed Python objects, querying them, persisting changes, and managing relationships between them. Everything in `django/db/models/` belongs to this domain.

The package is organized into several key subsystems:

- **Model base class** (`base.py`): The `ModelBase` metaclass and `Model` class. `ModelBase` intercepts class creation to register models with the app registry, attach `Options` metadata (`_meta`), auto-create exception classes (`DoesNotExist`, `MultipleObjectsReturned`, `NotUpdated`), and wire up fields through `contribute_to_class()`.
- **QuerySet and Manager** (`query.py`, `manager.py`): `QuerySet` is the public query API (filter, exclude, annotate, aggregate, etc.). `Manager` is the interface attached to model classes (default: `objects`) that proxies to a `QuerySet`. `BaseManager` supports `from_queryset()` for custom manager creation.
- **Fields** (`fields/`): All field types (CharField, IntegerField, ForeignKey, ManyToManyField, JSONField, CompositePrimaryKey, GeneratedField, etc.) and their related descriptors for attribute access.
- **Expressions** (`expressions.py`): `F`, `Q`, `Value`, `Case`/`When`, `Subquery`, `Exists`, `Window`, and the full expression tree used for annotations, aggregations, and complex lookups.
- **SQL compilation** (`sql/`): `Query` (internal query representation), `SQLCompiler` (turns queries into SQL), `WhereNode` (WHERE clause tree).
- **Deletion** (`deletion.py`): Cascade strategies (`CASCADE`, `PROTECT`, `RESTRICT`, `SET_NULL`, `SET_DEFAULT`, `DO_NOTHING`, `DB_CASCADE`, `DB_SET_NULL`, `DB_SET_DEFAULT`) and the `Collector` class that resolves deletion graphs.
- **Options** (`options.py`): The `Options` class (accessible as `Model._meta`) that stores all model metadata: fields, relations, ordering, constraints, indexes, db_table, permissions, etc.
- **Signals** (`signals.py`): `pre_init`, `post_init`, `pre_save`, `post_save`, `pre_delete`, `post_delete`, `m2m_changed`, `pre_migrate`, `post_migrate`. Uses `ModelSignal` which supports lazy sender resolution by string.
- **Constraints** (`constraints.py`): `CheckConstraint`, `UniqueConstraint` with deferrable support and violation error customization.
- **Indexes** (`indexes.py`): `Index` class for database index definitions.
- **Enums** (`enums.py`): `Choices`, `IntegerChoices`, `TextChoices` for field choice enumerations with auto-labeling.
- **Fetch modes** (`fetch_modes.py`): `FETCH_ONE`, `FETCH_PEERS`, `RAISE` controlling deferred field loading behavior.
- **Lookups** (`lookups.py`): `Lookup` and `Transform` base classes for field lookups (exact, contains, gt, etc.).

## Key behaviors

- **Metaclass-driven registration**: When a `Model` subclass is defined, `ModelBase.__new__()` processes `Meta` options, creates `_meta` (Options), attaches fields via `contribute_to_class()`, and registers the model with the app registry. Abstract models are not registered.
- **Three inheritance modes**: Abstract (no table, fields copied to children), multi-table (each model gets its own table, joined via implicit OneToOneField), and proxy (same table, different Python class with its own manager/methods).
- **Save with deferred field optimization**: `Model.save()` automatically detects deferred fields and converts to an `update_fields` save to avoid overwriting columns that were never loaded.
- **Async support**: Core operations (`save`, `delete`, `refresh_from_db`) have async variants (`asave`, `adelete`, `arefresh_from_db`) via `sync_to_async`. QuerySet provides async iteration and async versions of evaluation methods.
- **Validation pipeline**: `full_clean()` runs `clean_fields()`, `clean()`, `validate_unique()`, and optionally `validate_constraints()`. This is NOT called by `save()` -- it must be invoked explicitly or through ModelForm validation.
- **Deletion collector**: `Collector` traverses the relation graph to gather all objects affected by a deletion, respecting `on_delete` handlers. `DB_CASCADE`, `DB_SET_NULL`, `DB_SET_DEFAULT` delegate enforcement to the database.
- **Manager auto-creation**: If no manager is defined, Django adds a default `objects = Manager()`. Custom managers can coexist; the first defined manager becomes the default manager. Managers marked `use_in_migrations = True` are serialized into migration files.
- **Composite primary keys**: `CompositePrimaryKey` field allows multi-column primary keys (relatively new feature).
- **Generated fields**: `GeneratedField` maps to database-level generated/computed columns.
- **QuerySet chaining**: All QuerySet methods return new QuerySet instances (immutable pattern). Evaluation is lazy until iteration, `len()`, `bool()`, slicing with step, or `repr()`.

## Domain interactions

- **[Base](../base/_overview.md)**: Tests for basic model operations (create, save, delete, refresh) live in `tests/base/`.
- **[Fields](../fields/_overview.md)**: Field types are the building blocks of models. The fields subsystem (`django/db/models/fields/`) is tightly coupled.
- **[Constraints](../constraints/_overview.md)**: Constraint classes are used in `Meta.constraints` and validated during `full_clean()`.
- **[Indexes](../indexes/_overview.md)**: Index definitions used in `Meta.indexes`, compiled during migration.
- **[Expressions](../expressions/_overview.md)**: Expression tree used by QuerySet methods (annotate, filter with F/Q, aggregate).
- **[Lookups](../lookups/_overview.md)**: Lookup/Transform classes registered on fields, resolved during query compilation.
- **[Options](../options/_overview.md)**: `Options` class stores all model metadata and is central to introspection.
- **[Django](../django/_overview.md)**: The models package depends on `django.apps` (app registry), `django.db` (connections, router, transaction), `django.core.exceptions`, and `django.dispatch` (signals).
- **[Apps](../apps/_overview.md)**: Model registration happens through the app registry (`django.apps.registry.Apps`). The `tests/apps/models.py` file specifically tests model registration with custom `Apps` instances.
- **[Proxy](../proxy/_overview.md)**: Proxy model inheritance is handled by the metaclass and Options.
- **[Multi-table](../multi-table/_overview.md)**: Multi-table inheritance creates implicit parent links.

## Gotchas and edge cases

- **`save()` does not call `full_clean()`**: This is a longstanding Django design decision. Validation must be triggered explicitly. Model instances can be saved with invalid data if validation is bypassed.
- **Deferred fields and `save()`**: If a model is loaded with `only()` or `defer()`, saving it will only update the loaded fields (automatic `update_fields` optimization). This prevents accidental data loss but can be surprising if you set a deferred field and expect it to save.
- **`force_insert` vs `force_update` are mutually exclusive**: Passing both raises `ValueError`.
- **Manager ordering matters**: The first manager defined on a model class becomes `_default_manager`, which is used by admin, dumpdata, and other framework features. Reordering managers changes framework behavior.
- **Related name clashes with managers**: Fields E348 system check detects clashes between `related_name` and model manager names (recently enhanced in commit `fcf916884d`).
- **`on_delete=CASCADE` with non-nullable FK**: If the FK is not nullable and the database cannot defer constraint checks, Django sets the FK to NULL before deleting. This is a workaround for databases that check constraints immediately.
- **Custom `Apps` registry**: Models can be registered with a custom `Apps` instance (as demonstrated in `tests/apps/models.py` with `SoAlternative`), making them invisible to the default registry. This is used in testing and migration scenarios.
- **Signal sender lazy resolution**: `ModelSignal.connect()` accepts string senders (`"app_label.ModelName"`) and resolves them lazily via the app registry. This avoids circular import issues.
- **`FETCH_PEERS` deferred loading**: When using `FETCH_PEERS` fetch mode, accessing a deferred field on one instance triggers loading for all peer instances from the same QuerySet, reducing N+1 queries.

## Dependencies

- [Base](../base/_overview.md)
- [Constraints](../constraints/_overview.md)
- [Django](../django/_overview.md)
- [Expressions](../expressions/_overview.md)
- [Fields](../fields/_overview.md)
- [Indexes](../indexes/_overview.md)
- [Lookups](../lookups/_overview.md)
- [Options](../options/_overview.md)

## Referenced by

- [Apps](../apps/_overview.md)
- [Base](../base/_overview.md)
- [Constraints](../constraints/_overview.md)
- [Default-related-name](../default-related-name/_overview.md)
- [Expressions](../expressions/_overview.md)
- [Fields](../fields/_overview.md)
- [Indexes](../indexes/_overview.md)
- [Invalid-models](../invalid-models/_overview.md)
- [Lookups](../lookups/_overview.md)
- [Multi-table](../multi-table/_overview.md)
- [Options](../options/_overview.md)
- [Proxy](../proxy/_overview.md)
- [Tablespaces](../tablespaces/_overview.md)
