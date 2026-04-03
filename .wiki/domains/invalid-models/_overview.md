---
domain: invalid-models
confidence: llm-analyzed
last_updated: 2026-04-03
related_files:
  - tests/invalid_models_tests/__init__.py
  - tests/invalid_models_tests/test_models.py
  - tests/invalid_models_tests/test_ordinary_fields.py
  - tests/invalid_models_tests/test_relative_fields.py
  - tests/invalid_models_tests/test_deprecated_fields.py
  - tests/invalid_models_tests/test_custom_fields.py
  - tests/invalid_models_tests/test_backend_specific.py
  - tests/auth_tests/models/invalid_models.py
---

# Invalid-models

## What this domain does

This domain is Django's test suite for the **model system checks framework** (`django.core.checks`). It validates that Django correctly detects and reports configuration errors, warnings, and misuses when defining models, fields, relations, indexes, constraints, and Meta options. The tests do not define persistent models -- they use `@isolate_apps` to create throwaway model classes with intentional mistakes, then assert that `Model.check()` or `field.check()` returns the expected `Error` / `Warning` objects with correct error IDs (e.g., `models.E010`, `fields.E100`, `fields.E300`).

A secondary component lives in `tests/auth_tests/models/invalid_models.py`, which defines `CustomUserNonUniqueUsername` -- a deliberately non-standard `AbstractBaseUser` subclass used by `auth_tests` to verify that Django's auth system checks flag a non-unique `USERNAME_FIELD`.

## Key behaviors

- **Model-level Meta checks** (`test_models.py`, ~3100 lines): validates `unique_together`, `index_together`, `indexes`, `constraints`, `ordering`, `default_manager_name`, `db_table_comment`, multiple `AutoField` detection, `JSONField` checks, and lazy reference resolution.
- **Ordinary field checks** (`test_ordinary_fields.py`, ~1500 lines): validates field-specific rules for `AutoField`, `CharField`, `DecimalField`, `FilePathField`, `GenericIPAddressField`, `ImageField`, `IntegerField`, `BinaryField`, `FileField`, and others -- covering `max_length`, `choices`, `default`, `validators`, `db_index` interactions, and deprecated parameter usage.
- **Relational field checks** (`test_relative_fields.py`, ~2500 lines): validates `ForeignKey`, `ManyToManyField`, `OneToOneField` configuration -- missing targets, accessor/reverse query name clashes, self-referential relation clashes, `through` model field specification, `symmetrical` M2M, `related_name` conflicts with managers, and `on_delete` with database-level enforcement.
- **Deprecated field checks** (`test_deprecated_fields.py`): ensures removed fields like `IPAddressField`, `CommaSeparatedIntegerField`, and `NullBooleanField` produce errors when used outside of historical migrations.
- **Custom field checks** (`test_custom_fields.py`): verifies that fields with `db_type() -> None` (virtual columns) pass validation even when column name conflicts might otherwise occur.
- **Backend-specific checks** (`test_backend_specific.py`): confirms that `connection.validation.check_field()` is called so database backends can inject their own field validation rules.
- **Auth invalid model** (`auth_tests/models/invalid_models.py`): `CustomUserNonUniqueUsername` is used in `auth_tests/test_checks.py` and `auth_tests/test_management.py` to test that the auth checks framework warns about a non-unique `USERNAME_FIELD` and that `createsuperuser` still works when backed by a compatible authentication backend.

## Domain interactions

- **`django.core.checks`**: this domain is the primary consumer and exerciser of the model checks subsystem. Every test ultimately calls into `check()` methods that live in `django.db.models.base`, `django.db.models.fields`, and `django.db.models.fields.related`.
- **`auth_tests`**: `CustomUserNonUniqueUsername` is imported and used by `auth_tests.test_checks` and `auth_tests.test_management` to verify auth-specific system checks (e.g., `auth.W004`, `auth.E003`).
- **`django.db.models`**: the tests exercise the full model definition surface -- `Meta` options, field types, relation descriptors, managers, indexes, and constraints.
- **Database backends**: `test_backend_specific.py` tests that backend-specific validation hooks are invoked, touching `connection.validation`.

## Gotchas and edge cases

- All model classes in `tests/invalid_models_tests/` are created inside test methods using `@isolate_apps("invalid_models_tests")`. They are never migrated or persisted to a database. Some test classes inherit from `TestCase` (rather than `SimpleTestCase`) because certain checks query database capabilities at runtime (e.g., `JSONField` support, constraint features).
- `CustomUserNonUniqueUsername` in `auth_tests/models/` is a real registered model (imported in `__init__.py`), not isolated. It must exist in the app registry for auth system checks to reference it via `AUTH_USER_MODEL`.
- The error IDs follow a strict convention (`models.Exxxx`, `fields.Exxxx`, `fields.Wxxxx`). Tests assert exact IDs, so any renumbering in Django's checks framework will break these tests.

## Dependencies

- [Base](../base/_overview.md)
- [Constraints](../constraints/_overview.md)
- [Django](../django/_overview.md)
- [Expressions](../expressions/_overview.md)
- [Indexes](../indexes/_overview.md)
- [Lookups](../lookups/_overview.md)
- [Options](../options/_overview.md)

## Referenced by

- `tests/auth_tests/test_checks.py` -- imports and uses `CustomUserNonUniqueUsername`
- `tests/auth_tests/test_management.py` -- imports and uses `CustomUserNonUniqueUsername`
