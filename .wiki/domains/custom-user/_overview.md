---
domain: custom-user
confidence: llm-analyzed
last_updated: 2026-04-03
related_files:
  - tests/auth_tests/models/custom_user.py
  - tests/auth_tests/urls_custom_user_admin.py
  - tests/auth_tests/models/__init__.py
---

# Custom-user

## What this domain does

Defines custom user model variants used throughout Django's auth test suite. These models exercise Django's swappable `AUTH_USER_MODEL` mechanism by providing alternative user implementations that differ from the default `auth.User` in key ways: different username fields, different required fields, missing standard fields, and non-standard primary keys. They are the backbone of testing that Django's auth framework (forms, views, management commands, backends, admin) works correctly with any custom user model, not just the built-in one. `[llm-analyzed]`

## Key behaviors

- **CustomUser**: Uses `email` as `USERNAME_FIELD` instead of `username`. Requires `date_of_birth` and `first_name`. Does not use `PermissionsMixin`; instead implements permission methods (`has_perm`, `has_module_perms`, etc.) as hardcoded stubs returning `True`. Uses a custom manager named `custom_objects` (not `objects`). Exposes `is_staff` as a property derived from `is_admin`.

- **CustomUserManager**: Extends `BaseUserManager` with `create_user`, `acreate_user` (async), and `create_superuser`. Enforces that email is provided and normalizes it. The async variant uses `await user.asave()`.

- **ExtensionUser**: Extends `AbstractUser` (the full built-in user with permissions). Adds a `date_of_birth` required field. Instantiated inside a `RemoveGroupsAndPermissions` context manager to avoid M2M `related_name` clashes with the default `AbstractUser` groups/permissions fields.

- **RemoveGroupsAndPermissions**: Context manager that temporarily replaces the `groups` and `user_permissions` M2M fields on `AbstractUser` and `PermissionsMixin` with blank versions (no `related_name`). This prevents reverse accessor clashes when multiple models inherit from `AbstractUser` in the same app.

- **CustomUserWithoutIsActiveField**: Inherits `AbstractBaseUser` but intentionally omits the `is_active` field. Uses `username` as `USERNAME_FIELD` and the default `UserManager`. Used to test that auth backends handle missing `is_active` gracefully.

- **CustomUserCompositePrimaryKey**: Inherits `AbstractBaseUser` with a `CompositePrimaryKey("email", "date_of_birth")`. Tests that password reset and other auth views work with composite PKs.

## Domain interactions

- **test_views.py**: `CustomUser` and `CustomUserCompositePrimaryKey` are used in password reset view tests via `@override_settings(AUTH_USER_MODEL=...)`.
- **test_forms.py**: `CustomUserWithoutIsActiveField` is used to test `UserCreationForm` when `is_active` is absent. `ExtensionUser` tests form behavior with extended user models.
- **test_auth_backends.py**: `CustomUserWithoutIsActiveField` tests that `ModelBackend` handles users without `is_active`. `CustomUser` is used in `RowlevelBackendTest` and custom backend tests. `ExtensionUser` tests backend behavior with extended models.
- **test_management.py**: `CustomUser` tests `createsuperuser` and `changepassword` management commands with non-standard username fields and required fields.
- **test_checks.py**: Uses custom user models from `invalid_models.py` (not this file) to test system checks like `auth.E003`, `auth.W004`.
- **test_handlers.py**: `CustomUser` tests `WSGI` handler with custom user model.
- **test_templates.py**: Tests template rendering with `CustomUser` as the auth user.
- **test_basic.py**: Tests basic user creation with `CustomUser`.
- **urls_custom_user_admin.py**: Registers the current `AUTH_USER_MODEL` in a custom admin site with a `CustomUserAdmin` that patches the PK for `LogEntry` compatibility (workaround for UUID PKs).

## Gotchas and edge cases

- **`custom_objects` manager name**: `CustomUser` and `CustomUserCompositePrimaryKey` use `custom_objects` instead of the default `objects`. Code using `CustomUser.objects` will fail with `AttributeError`.
- **RemoveGroupsAndPermissions is module-level**: `ExtensionUser` is defined inside a `with RemoveGroupsAndPermissions()` block at module load time. This means importing `custom_user.py` temporarily mutates `AbstractUser._meta`. If imports happen in an unexpected order or are interrupted, `AbstractUser`'s M2M fields could be left in an inconsistent state.
- **Hardcoded permission stubs**: `CustomUser.has_perm()` always returns `True`, so tests using `CustomUser` cannot meaningfully test permission denial scenarios.
- **No `objects` manager on CustomUser**: The default manager is `custom_objects`, not `objects`. This is intentional to test that Django does not hardcode the manager name.
- **Composite PK model**: `CustomUserCompositePrimaryKey` tests a relatively new Django feature. Password reset token generation and URL routing must handle composite PKs correctly.

## Dependencies

- [Base](../base/_overview.md) -- `AbstractBaseUser`, `AbstractUser`, `BaseUserManager`
- [Django](../django/_overview.md) -- `django.contrib.auth.models`, `django.db.models`
- [Constraints](../constraints/_overview.md)
- [Expressions](../expressions/_overview.md)
- [Indexes](../indexes/_overview.md)
- [Lookups](../lookups/_overview.md)
- [Options](../options/_overview.md)

## Referenced by

- `tests/auth_tests/test_views.py` -- password reset tests with custom user models
- `tests/auth_tests/test_forms.py` -- user creation/change form tests
- `tests/auth_tests/test_auth_backends.py` -- authentication backend tests
- `tests/auth_tests/test_management.py` -- `createsuperuser` / `changepassword` tests
- `tests/auth_tests/test_basic.py` -- basic user model tests
- `tests/auth_tests/test_handlers.py` -- WSGI handler tests
- `tests/auth_tests/test_templates.py` -- template rendering tests
- `tests/auth_tests/test_checks.py` -- system check tests
- `tests/auth_tests/urls_custom_user_admin.py` -- custom admin site registration
- `tests/auth_tests/models/__init__.py` -- re-exports all models from this file
