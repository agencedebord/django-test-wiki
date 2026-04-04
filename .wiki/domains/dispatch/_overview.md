---
domain: dispatch
confidence: llm-analyzed
last_updated: 2026-04-03
related_files:
  - django/dispatch/dispatcher.py
  - django/dispatch/license.txt
---

# Dispatch

## What this domain does
Django's signal/dispatch system — a pub/sub mechanism enabling decoupled communication between components. Forked from PyDispatcher, it allows senders to broadcast events and receivers to subscribe without either knowing about the other. The core abstraction is `Signal`: instances are declared globally, receivers connect to them, and senders fire them with `send()`. This is how Django internally notifies app code of framework lifecycle events (model save, request start, etc.) without hard-coding callbacks. `[llm-analyzed]`

## Key behaviors
- **Weak references by default — receivers can be GC'd without explicit disconnect**: When `weak=True` (the default), receivers are stored as `weakref.ref` or `weakref.WeakMethod` (for bound methods). If the receiver object is garbage collected, a callback flags `_dead_receivers=True` and the dead entry is cleaned up on the next `connect()`, `disconnect()`, or `send()`. This means you don't need to manually disconnect receivers whose owning objects are destroyed — but it also means a lambda or locally-defined function passed as receiver will be immediately collected unless you hold a strong reference to it yourself. `[llm-analyzed]`
- **Async receivers are supported and run in parallel**: `iscoroutinefunction(receiver)` is checked at connect time and the `is_async` flag is stored per-receiver. When `send()` fires, sync and async receivers are separated. Async receivers are gathered and executed in parallel via `asyncio.TaskGroup` inside `_run_parallel()`. Sync receivers are run sequentially (or wrapped via `sync_to_async` when called from an async context). This parallelism means async receivers for the same signal fire concurrently — they must not assume ordering relative to each other. `[llm-analyzed]`
- **contextvars are propagated and restored across async tasks**: `_run_parallel` copies the current `contextvars.Context` before launching tasks, and `_restore_context` pushes any changed values back to the calling context after all tasks finish. This ensures that context variables set by async receivers (e.g. request-scoped state) propagate downstream, but it also means a receiver can inadvertently mutate context visible to other code after the signal fires. `[llm-analyzed]`
- **dispatch_uid prevents duplicate receiver registration**: If `dispatch_uid` is provided, the lookup key becomes `(dispatch_uid, id(sender))` instead of `(id(receiver), id(sender))`. A receiver is only appended if no entry with that key already exists. This is the mechanism Django uses internally to ensure signal handlers registered at module import time (e.g. via `@receiver` decorator in `apps.py`) are not re-registered on app reload in development. `[llm-analyzed]`
- **Sender-scoped subscriptions filter delivery**: Receivers can be connected to a specific sender or to `None` (any sender). At send time, only receivers matching the actual sender or connected to `None` are called. The sender is also stored as a weakref when possible — if the sender object is garbage collected, its associated receivers are cleaned up, preventing id reuse collisions between distinct short-lived sender objects. `[llm-analyzed]`

## Domain interactions
- **Conf**: Imports `django.conf.settings` lazily inside `connect()` to check `settings.configured` and `settings.DEBUG`. The lazy import avoids a circular dependency during Django's bootstrap, since `conf` itself may trigger signal sends before settings are fully loaded. `[llm-analyzed]`
- **Utils**: Uses `django.utils.inspect.func_accepts_kwargs` to validate that a receiver accepts `**kwargs`. This is a hard requirement — all receivers must be designed to ignore unknown keyword arguments, since senders may pass arbitrary extra context. `[llm-analyzed]`
- **Django (asgiref)**: Uses `asgiref.sync.async_to_sync` and `sync_to_async` to bridge sync/async boundary when mixing receiver types. This makes the dispatch layer ASGI-aware without owning the event loop management itself. `[llm-analyzed]`

## Gotchas and edge cases
- Lambdas and locally-scoped functions passed as receivers with weak=True (the default) will be immediately garbage collected and silently never called. Always hold a module-level or instance-level reference, or pass weak=False. `[llm-analyzed]`
- Async receivers for the same signal run in parallel (TaskGroup), not sequentially. If two async receivers modify shared state, they will race. `[llm-analyzed]`
- The sender cache (use_caching=True) is fully cleared on every connect/disconnect, not just for the affected sender. A system that dynamically connects receivers at runtime will defeat the cache entirely. `[llm-analyzed]`
- In non-DEBUG mode, connecting a receiver that doesn't accept **kwargs will not raise at connect time — the error will only appear when the signal is sent. `[llm-analyzed]`
- BaseExceptionGroup from parallel async receivers is partially unwrapped: if exactly one exception occurred, it is re-raised directly; if multiple exceptions occurred, the full ExceptionGroup propagates. Callers need to handle both cases. `[llm-analyzed]`

## Dependencies
- [Conf](../conf/_overview.md)
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
- [Tasks](../tasks/_overview.md)
- [Templates](../templates/_overview.md)
- [Templatetags](../templatetags/_overview.md)
- [Tests](../tests/_overview.md)
- [Utils](../utils/_overview.md)