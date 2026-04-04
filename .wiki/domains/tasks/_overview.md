---
domain: tasks
confidence: llm-analyzed
last_updated: 2026-04-03
related_files:
  - django/tasks/signals.py
  - django/tasks/checks.py
  - django/tasks/backends/immediate.py
  - django/tasks/backends/dummy.py
  - django/tasks/backends/base.py
  - django/tasks/__init__.py
  - django/tasks/exceptions.py
  - django/tasks/base.py
---

# Tasks

## What this domain does
Django's built-in background task system. It provides a pluggable backend abstraction for enqueueing and executing functions asynchronously, following the same connection-handler pattern as Django databases. The domain defines the task lifecycle (enqueue → run → result), enforces invariants at decoration time, and ships two concrete backends: `DummyBackend` (in-memory, for testing) and `ImmediateBackend` (synchronous inline execution, for development/simple cases). Actual async worker backends (database-backed, queue-backed) are expected to be provided externally and registered under `s... `[llm-analyzed]`

## Key behaviors
- **Task validation happens at decoration time, not at enqueue time**: When `@task` is applied to a function, `Task.__post_init__` immediately calls `backend.validate_task(self)`. This means configuration errors (wrong queue name, async task on non-async backend, invalid priority) raise `InvalidTask` at import time, not when the task is called. This is intentional: fail fast so misconfigured tasks are caught during startup/testing rather than silently failing at runtime. `[llm-analyzed]`
- **Tasks must be module-level functions — closures, lambdas, and methods are rejected**: `BaseTaskBackend.validate_task` calls `is_module_level_function(task.func)` and raises `InvalidTask` if it returns False. This is a hard constraint, not a limitation of specific backends. The reason is serialization: task functions must be importable by path (`module_path` property uses `func.__module__` + `func.__qualname__`), which is impossible for closures or bound methods. `[llm-analyzed]`
- **Priority is a bounded integer in [-100, 100] — floats and out-of-range values are rejected**: `validate_task` checks `int(task.priority) != task.priority` (rejects floats), and enforces the [-100, 100] range. Additionally, if `backend.supports_priority` is False, any non-zero priority raises `InvalidTask`. Default priority is 0. `[llm-analyzed]`
- **Context injection: if takes_context=True, the function must declare 'context' as its first argument**: This is validated via `get_func_args`. The convention is strict: the argument must be named exactly `context`, not just be present. `ImmediateBackend` passes a `TaskContext(task_result=task_result)` as the first argument when executing. This lets tasks introspect their own result ID, status, etc. at runtime. `[llm-analyzed]`
- **TaskResult and Task are frozen dataclasses, but ImmediateBackend bypasses this with object.__setattr__**: Both dataclasses are declared `frozen=True`. However, `ImmediateBackend._execute_task` mutates `task_result` throughout execution (setting `enqueued_at`, `status`, `started_at`, `_return_value`, etc.) by calling `object.__setattr__` directly. This is the intentional pattern for backends: the frozen constraint prevents accidental external mutation, but the backend that owns the result is allowed to update it. `[llm-analyzed]`

## Domain interactions
- **Django.utils.connection**: TaskBackendHandler extends BaseConnectionHandler and ConnectionProxy — the exact same pattern Django uses for database connections. This means task backends benefit from lazy initialization, per-alias configuration, and settings-override support in tests. `[llm-analyzed]`
- **Django.core.checks**: Registers a system check via checks.py that iterates all configured backends and calls backend.check(). This runs during `manage.py check` and on server startup, surfacing backend misconfiguration early. `[llm-analyzed]`
- **Django.core.signals / django.dispatch**: Emits three lifecycle signals: task_enqueued, task_started, task_finished. Default handlers log at DEBUG/INFO/ERROR level. The setting_changed receiver clears backend connections when TASKS is overridden (enabling test isolation with @override_settings). `[llm-analyzed]`
- **Django.utils.json**: normalize_json is called on task args and return values to enforce JSON round-trip compatibility. This is a shared contract between the task domain and any backend that serializes tasks to persistent storage. `[llm-analyzed]`
- **Asgiref**: Task.call/acall use async_to_sync and sync_to_async to bridge sync/async execution. BaseTaskBackend.aenqueue wraps enqueue with sync_to_async(thread_sensitive=True) so async callers don't need separate implementations unless they want true async behavior. `[llm-analyzed]`
- **External backends (consumers of this domain)**: External backends subclass BaseTaskBackend and declare capability flags (supports_defer, supports_async_task, supports_get_result, supports_priority). They set task_class to a Task subclass if needed. They are responsible for implementing enqueue() and optionally get_result(). The base class provides aenqueue() and aget_result() as sync wrappers automatically. `[llm-analyzed]`

## Gotchas and edge cases
- DummyBackend.worker_ids is a mutable list on a frozen dataclass — both DummyBackend and ImmediateBackend call `.append()` on it directly. The frozen constraint only prevents re-assignment of the attribute, not mutation of the list itself. `[llm-analyzed]`
- ImmediateBackend generates a single worker_id at `__init__` time (via `get_random_string(32)`) and reuses it for all tasks executed by that backend instance. Multiple tasks on the same backend share the same worker_id. `[llm-analyzed]`
- _return_value is declared with `field(init=False, default=None)` — it is never passed to the constructor, only set post-execution via `object.__setattr__`. Code that accesses `task_result._return_value` on a DummyBackend result (which is never executed) will always get `None`. `[llm-analyzed]`
- Queue validation is only enforced when `self.queues` is non-empty. If `QUEUES` is not specified in the backend params, `self.queues` is `{DEFAULT_TASK_QUEUE_NAME}` (defaults to `{'default'}`), so any other queue name will fail validation. An empty queues set would allow any queue name, but the default constructor always includes at least the default queue. `[llm-analyzed]`
- The `using()` method on `Task` creates a new Task via `dataclasses.replace`, which triggers `__post_init__` again — meaning `validate_task` is called again with the new parameters. Changing to an incompatible backend via `task.using(backend='other')` may raise `InvalidTask`. `[llm-analyzed]`

## Dependencies
- [Base](../base/_overview.md)
- [Conf](../conf/_overview.md)
- [Core](../core/_overview.md)
- [Db](../db/_overview.md)
- [Dispatch](../dispatch/_overview.md)
- [Django](../django/_overview.md)
- [Utils](../utils/_overview.md)

## Referenced by
- [Base](../base/_overview.md)
- [Conf](../conf/_overview.md)
- [Contrib](../contrib/_overview.md)
- [Core](../core/_overview.md)
- [Db](../db/_overview.md)
- [Django](../django/_overview.md)
- [Django-views](../django-views/_overview.md)
- [Templatetags](../templatetags/_overview.md)
- [Tests](../tests/_overview.md)
- [Utils](../utils/_overview.md)