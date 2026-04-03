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
The `options` domain implements Django's model metadata system — the `Options` class that powers `model._meta`. It is the single source of truth for every configuration decision a model carries: its database table name, field lists, manager names, ordering, permissions, proxy relationships, and more. Every model in the project attaches one `Options` instance to itself at class-creation time via `contribute_to_class`. This domain has no project-level dependencies; it is a pure foundational layer that every other model-bearing domain consumes. `[llm-analyzed]`

## Key behaviors
- **Recognizes exactly the Meta options listed in DEFAULT_NAMES**: Only the options enumerated in the `DEFAULT_NAMES` tuple are read from a model's inner `class Meta` and applied to the `Options` instance. Any attribute not in this list is silently ignored during `contribute_to_class`. Adding a new Meta option to Django requires adding it to `DEFAULT_NAMES` first. `[llm-analyzed]`
- **Derives db_table automatically from app label and class name**: If `db_table` is not set, Django generates it as `{app_label}_{model_name}` where `model_name` is the lowercase class name. This happens in `contribute_to_class` after `object_name` and `model_name` are resolved from the class. Overriding `db_table` entirely bypasses this derivation. `[llm-analyzed]`
- **Normalizes unique_together to always be a tuple of tuples**: `normalize_together` accepts either a flat tuple of strings (e.g. `('field_a', 'field_b')`) representing a single constraint, or a tuple of tuples for multiple constraints. It normalizes both forms to `((field_a, field_b),)` so downstream code only has to handle one shape. Invalid values are returned verbatim and caught later by the system check framework. `[llm-analyzed]`
- **Wraps field lists in ImmutableList to prevent accidental mutation**: `make_immutable_fields_list` wraps any returned field collection in an `ImmutableList` that emits a warning if mutated. This exists because `FORWARD_PROPERTIES` and `REVERSE_PROPERTIES` are cached; mutating the list in place would corrupt the cache for every caller. The warning names the property, making the source of the mutation easy to trace. `[llm-analyzed]`
- **Separates forward and reverse cached properties for targeted cache invalidation**: `FORWARD_PROPERTIES` (fields, managers, concrete_fields, etc.) are properties whose answers are determinable from the model itself. `REVERSE_PROPERTIES` (related_objects, fields_map, _relation_tree) depend on other models pointing back. They are invalidated separately so that adding a ForeignKey on model B pointing to model A only busts A's reverse cache, not its forward cache. `[llm-analyzed]`

## Domain interactions
- **All model-bearing domains (article, bar, base, etc.)**: Every model class that defines `class Meta` gets an `Options` instance attached as `cls._meta` via `contribute_to_class`. All ORM operations (queries, migrations, admin, serialization) consult `_meta` for field lists, table names, ordering, managers, and constraints. This domain is a write-once-at-class-creation, read-many-at-runtime registry. `[llm-analyzed]`
- **Django.apps (AppRegistry)**: `Options` holds a reference to the app registry (`self.apps`, defaulting to the global `apps`). This allows models in isolated registries (e.g. test setups) to resolve their app config without touching the global state. The `app_config` property looks up by `app_label` without triggering imports. `[llm-analyzed]`
- **Django.db.connections / django backends**: `contribute_to_class` imports `connection` and `truncate_name` from the DB backend to handle database-specific table name constraints (notably Oracle's 30-character limit). The `db_tablespace` option is silently ignored by backends that don't support tablespaces. `[llm-analyzed]`
- **Django.core.signals (setting_changed)**: The `setting_changed` signal is imported, indicating that `Options` registers a handler to react when Django settings change at runtime (relevant in test suites using `@override_settings`). Cached properties in `FORWARD_PROPERTIES` and `REVERSE_PROPERTIES` need to be cleared when settings that affect field resolution change. `[llm-analyzed]`

## Gotchas and edge cases
- A model defined outside of any app in INSTALLED_APPS must explicitly set `app_label` in its Meta, otherwise Django cannot resolve it and will raise ImproperlyConfigured. `[llm-analyzed]`
- For Oracle, Django may silently shorten and uppercase table names to meet the 30-character limit. To preserve a specific casing, wrap the value in double-quotes: `db_table = '"my_table_name"'`. This quoting is harmless on other backends. `[llm-analyzed]`
- MySQL/MariaDB: always use lowercase `db_table` values. The MySQL backend is case-sensitive on some filesystems and the convention is enforced by recommendation, not by Django code. `[llm-analyzed]`
- unique_together accepts a flat tuple like `('a', 'b')` and normalizes it to `(('a', 'b'),)` — one constraint on two fields. A tuple of tuples like `(('a', 'b'), ('c', 'd'))` is two separate constraints. The normalization is invisible; passing the wrong shape produces the wrong constraints silently. `[llm-analyzed]`
- The NOTE comment 'We can't modify a dictionary's contents while looping' signals a pattern somewhere in this file where iteration over a dict must be done over a snapshot (e.g. `list(d.items())`). Callers adding fields or managers during model construction must be aware of this ordering constraint. `[llm-analyzed]`

## Notes from code
- [NOTE] We can't modify a dictionary's contents while looping

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