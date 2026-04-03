---
domain: custom-permissions
confidence: llm-analyzed
last_updated: 2026-04-03
related_files:
  - tests/auth_tests/models/custom_permissions.py
  - tests/auth_tests/test_auth_backends.py
  - tests/auth_tests/models/__init__.py
---

# Custom-permissions

## What this domain does

This domain defines `CustomPermissionsUser`, a test-only custom user model that uses **email as the unique identifier** while relying on Django's standard `PermissionsMixin` for permissions handling. It exists to verify that `PermissionsMixin` provides everything needed to work correctly with `ModelBackend` when applied to a non-default user model.

Unlike the `CustomUser` model (which manually implements permission methods like `has_perm`, `get_all_permissions`, etc.), `CustomPermissionsUser` inherits them from `PermissionsMixin`, making it the canonical test for the mixin-based permissions approach.

## Key behaviors

- **Extends `AbstractBaseUser` + `PermissionsMixin`**: does not extend `AbstractUser`, so it only gets authentication and permissions -- no `username`, `first_name`, `last_name`, or `is_staff` fields.
- **Uses email as `USERNAME_FIELD`**: authenticates users by email address instead of username.
- **Requires `date_of_birth`**: listed in `REQUIRED_FIELDS`, enforced during `create_user` / `create_superuser`.
- **Manager named `custom_objects`** (not `objects`): the manager is `CustomPermissionsUserManager`, which extends `CustomUserManager` from the `custom-user` domain, adding a `create_superuser` that sets `is_superuser = True`.
- **Defined inside `RemoveGroupsAndPermissions` context manager**: this temporarily replaces the `groups` and `user_permissions` M2M fields on `PermissionsMixin` and `AbstractUser` to avoid `related_name` clashes when multiple user models coexist in the test database.
- **`is_superuser` controls all permissions**: since permissions come from `PermissionsMixin`, superuser status grants all permissions via the standard Django mechanism.

## Domain interactions

- **custom-user**: imports `CustomUserManager` (base manager) and `RemoveGroupsAndPermissions` (context manager to avoid M2M related_name conflicts).
- **django.contrib.auth**: uses `AbstractBaseUser` and `PermissionsMixin` from the auth framework.
- **test_auth_backends**: the `CustomPermissionsUserModelBackendTest` class sets `AUTH_USER_MODEL = "auth_tests.CustomPermissionsUser"` and runs the full `BaseModelBackendTest` suite (permission checks, superuser behavior, inactive user handling) against this model.

## Gotchas and edge cases

- **`RemoveGroupsAndPermissions` is required**: without this context manager, defining `CustomPermissionsUser` would cause Django system check errors (fields.E304 / E305) due to `related_name` clashes on `groups` and `user_permissions` with the default `User` model. This is a test infrastructure concern, not something needed in real projects (which would use `AUTH_USER_MODEL` to designate a single user model).
- **Manager is `custom_objects`, not `objects`**: code using `CustomPermissionsUser.objects` will fail. The tests use `CustomPermissionsUser._default_manager` to remain manager-name-agnostic.
- **No `is_staff` or `is_active` fields**: `AbstractBaseUser` does not provide `is_staff`. The model has no `is_active` field either, which means the default `is_active = True` behavior comes from `AbstractBaseUser`'s property. Tests that set `user.is_staff = True` add it dynamically.
- **Not a swappable user model at module level**: the `AUTH_USER_MODEL` override is applied per-test-class via `@override_settings`, so this model coexists with Django's default `User` in the test database schema.

## Dependencies

- [Custom-user](../custom-user/_overview.md) -- `CustomUserManager`, `RemoveGroupsAndPermissions`
- [Django](../django/_overview.md) -- `AbstractBaseUser`, `PermissionsMixin`, `ModelBackend`

## Referenced by

- `tests/auth_tests/test_auth_backends.py` -- `CustomPermissionsUserModelBackendTest`
- `tests/auth_tests/models/__init__.py` -- re-exported in `__all__`
