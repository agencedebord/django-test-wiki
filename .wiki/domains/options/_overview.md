---
domain: options
confidence: llm-analyzed
last_updated: 2026-04-03
related_files:
  - docs/ref/models/options.txt
  - django/db/models/options.py
---

# Options

## What this domain does
The `options` domain implements Django's `Model._meta` API — the `Options` class that stores and manages all metadata for a model class. It is the central registry for every model's structural configuration: field lists, table names, ordering, permissions, manager names, proxy relationships, and inheritance state. Every other domain in the project reads from `_meta` to understand what a model is and how it should behave. `[llm-analyzed]`

## Key behaviors
- **Centralizes model metadata resolution at class construction time**: `Options.contribute_to_class` is called during model metaclass processing and wires `cls._meta = self`. At that point it derives `object_name` (raw class name), `model_name` (lowercased), and a human-readable `verbose_name` via `camel_case_to_spaces`. All downstream code that asks 'what is this model called?' goes through `_meta`. `[llm-analyzed]`
- **Enforces immutability on derived field lists**: Field collections returned by the public API (e.g. `concrete_fields`, `many_to_many`) are wrapped in `ImmutableList` with an explicit warning message. This prevents callers from accidentally mutating cached state that is shared across the process. The NOTE comment in the source explicitly flags this: dict contents cannot be modified while looping — the same constraint applies here. `[llm-analyzed]`
- **Manages two separate caches: FORWARD_PROPERTIES and REVERSE_PROPERTIES**: `FORWARD_PROPERTIES` covers fields, managers, and computed field maps — derived from the model itself. `REVERSE_PROPERTIES` covers reverse relations and the relation tree — derived from other models pointing at this one. These two sets are invalidated separately because reverse relations are populated lazily after all apps are loaded, while forward properties are available immediately after the class is created. `[llm-analyzed]`
- **Derives default DB table name from app_label + model_name**: Unless `db_table` is set explicitly, Django computes the table name as `{app_label}_{model_name}`. This is a convention the framework enforces globally. Overriding requires setting `Meta.db_table`. For Oracle, Django additionally truncates and uppercases table names unless the name is explicitly quoted with double quotes inside the string value. `[llm-analyzed]`
- **Normalizes `unique_together` / `option_together` to always be a tuple of tuples**: `normalize_together()` handles the shorthand form where a developer writes a single flat tuple instead of a tuple-of-tuples. It converts `('a', 'b')` to `(('a', 'b'),)` so all downstream code can assume uniform structure. Invalid inputs are returned verbatim so the system check framework can report them with proper error messages rather than crashing at startup. `[llm-analyzed]`

## Domain interactions
- **Virtually all model-using domains**: Every domain that defines a model reads `_meta` to discover fields, table name, ordering, permissions, and manager configuration. `options` is a read-heavy, write-once domain: it is populated at class creation and then consumed everywhere else. `[llm-analyzed]`
- **Migrations / db**: The `managed`, `db_table`, `db_tablespace`, `indexes`, `constraints`, and `original_attrs` attributes are consumed by the migration framework to generate and apply schema changes. `managed=False` is the primary escape hatch when Django should not own the DDL. `[llm-analyzed]`
- **Managers (base_manager, default_manager)**: `Options` resolves manager names (`base_manager_name`, `default_manager_name`) and caches resolved manager instances in `FORWARD_PROPERTIES`. The manager lookup is delegated to `apps` registry but the resolution result lives on `_meta`. `[llm-analyzed]`
- **Proxy / abstract inheritance**: Abstract models set `abstract=True` and are never registered as concrete models. Proxy models set `proxy=True` and share the parent's table. `Options` tracks both flags and the resulting `proxy_for_model` chain, which is consumed by queryset construction and admin registration. `[llm-analyzed]`
- **Reverse relations / related_objects**: `REVERSE_PROPERTIES` (including `_relation_tree` and `fields_map`) are populated after all apps finish loading, not at class creation time. Code that accesses reverse relations before `AppRegistryNotReady` is resolved will get an empty or incorrect result. `[llm-analyzed]`

## Gotchas and edge cases
- Mutating any ImmutableList returned by `_meta` (e.g. `_meta.concrete_fields`) raises an error at runtime with a descriptive message, but only if the code actually tries to mutate it — silent bugs can appear if callers assume they got a plain list and skip mutation. `[llm-analyzed]`
- `managed=False` does NOT skip M2M join table creation when the other side is managed. Developers expecting full DDL isolation must explicitly use `through=` with their own managed intermediary model. `[llm-analyzed]`
- The `apps` attribute on `Options` can be overridden to point at a custom app registry. Most code assumes `django.apps.apps` is canonical, so using a custom registry can cause subtle failures in generic views, admin, and signals that do their own registry lookups. `[llm-analyzed]`
- `default_related_name` affects `related_query_name` as well. Setting it on an abstract model requires `%(app_label)s` and `%(model_name)s` interpolation placeholders to avoid name collisions across concrete subclasses — omitting these will cause a system check error. `[llm-analyzed]`
- `original_attrs` is populated during `contribute_to_class` from the raw `Meta` class. If a developer programmatically modifies `_meta` after class creation (e.g. in tests or dynamic model generation), `original_attrs` will not reflect those changes, causing migrations to serialize stale values. `[llm-analyzed]`

## Notes from code
- [NOTE] We can't modify a dictionary's contents while looping

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