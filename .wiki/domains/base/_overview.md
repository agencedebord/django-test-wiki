---
domain: base
confidence: llm-analyzed
last_updated: 2026-04-03
related_files:
  - tests/serializers/models/base.py
  - django/core/handlers/base.py
  - django/db/models/base.py
---

# Base

## What this domain does
The "base" domain provides the two foundational pillars of Django's runtime: the HTTP request/response pipeline (BaseHandler) and the ORM model class system (ModelBase metaclass + Model). These are not feature modules — they are the infrastructure everything else builds on. Almost every other domain in the codebase either inherits from Model or passes through BaseHandler. `[llm-analyzed]`

## Key behaviors
- **Middleware chain assembled in reverse at load time, not per-request**: load_middleware() iterates settings.MIDDLEWARE in reverse order, building a nested callable chain where each middleware wraps the next. The chain is stored in _middleware_chain once fully assembled. This means middleware startup cost is paid once at server start, not on every request. The assignment to _middleware_chain is intentionally the last step — it doubles as an initialization-complete flag checked by callers. `[llm-analyzed]`
- **Sync/async middleware negotiation at middleware load time**: Each middleware advertises its capabilities via sync_capable (defaults True) and async_capable (defaults False) class attributes. BaseHandler.adapt_method_mode() wraps sync methods with sync_to_async or async methods with async_to_sync using asgiref, depending on whether the handler chain is currently in async mode. Exception-handling middleware (process_exception) is always adapted to sync mode regardless of the overall handler mode — this is an explicit current limitation noted in the code. `[llm-analyzed]`
- **Separate async response path to avoid context-switch overhead**: get_response_async() exists as a distinct code path from get_response() rather than funneling through a single adapter. The comment explicitly says funneling WSGI through a single async get_response() is too slow. This is a deliberate performance trade-off that creates code duplication in exchange for eliminating unnecessary thread context switches for ASGI deployments. `[llm-analyzed]`
- **ModelBase metaclass intercepts class creation to wire Django ORM infrastructure**: ModelBase.__new__() runs on every class definition that inherits from Model. It separates attributes into those with contribute_to_class() (Django descriptors, fields, managers) and plain Python attributes. Plain attrs go directly to type.__new__() so __set_name__ works correctly. Django-aware attrs are contributed after the class exists, allowing them to register themselves on _meta. `[llm-analyzed]`
- **Per-model DoesNotExist, MultipleObjectsReturned, and NotUpdated exceptions are dynamically created**: ModelBase creates model-specific exception subclasses (e.g., MyModel.DoesNotExist) that inherit from their parent model's equivalent exceptions. This makes isinstance checks on parent models work correctly in multi-table inheritance hierarchies. NotUpdated also inherits from DatabaseError for backward compatibility — the comment notes that __subclasshook__ is not considered in exception handling. `[llm-analyzed]`

## Domain interactions
- **Options**: ModelBase instantiates Options(_meta) for every model class, passing the inner Meta class. Options is the central registry for field definitions, db_table, ordering, constraints, indexes, and proxy relationships. Everything the ORM knows about a model's schema lives in _meta. `[llm-analyzed]`
- **Fields**: Fields use contribute_to_class() to register themselves on _meta during ModelBase class creation. This is why fields must be declared at class level — ModelBase only processes them at class-definition time. `[llm-analyzed]`
- **Proxy**: Proxy model support flows through ModelBase: proxy models share their parent's db_table but get their own _meta, manager chain, and exception classes. The base_meta check in ModelBase drives inheritance of ordering/get_latest_by which matters especially for proxy models that want the parent's default ordering. `[llm-analyzed]`
- **Django (core handlers)**: BaseHandler is the shared base for both WSGIHandler and ASGIHandler. It provides load_middleware(), adapt_method_mode(), and the get_response/_get_response pair. Concrete handler subclasses call load_middleware() in their __call__ and then delegate to get_response() or get_response_async(). `[llm-analyzed]`
- **Expressions**: Model.save() and related operations use ExpressionWrapper, DatabaseDefault, and F/Q from expressions. The base model layer is aware of database-side default values (DatabaseDefault) and uses them during insert to skip sending Python-side values when the DB should generate them. `[llm-analyzed]`
- **Lookups**: LOOKUP_SEP ('__') is used in Model to traverse field paths during validation and deferred field resolution. The base model layer uses this constant but delegates actual lookup resolution to the lookup domain. `[llm-analyzed]`

## Gotchas and edge cases
- MiddlewareNotUsed rolls back the handler to adapted_handler (not the previous handler), not to the middleware's input. This means if middleware raises MiddlewareNotUsed, the already-adapted version of the previous handler becomes the new top of stack — the adaptation is not undone. `[llm-analyzed]`
- The _middleware_chain flag pattern means that if load_middleware() raises mid-way through, _middleware_chain stays None and the server will appear uninitialized. There is no partial-init recovery. `[llm-analyzed]`
- ModelBase runs at import time. Any side effect in contribute_to_class() — like querying the database for default values — will execute before Django's app registry is ready, likely causing AppRegistryNotReady errors. Fields and managers must be written to defer DB access. `[llm-analyzed]`
- The exception inheritance chain for DoesNotExist walks the MRO of the bases list to find parent models with _meta. If multiple abstract bases are in the inheritance chain, none of their DoesNotExist exceptions are included — only non-abstract parents contribute. `[llm-analyzed]`
- Non-abstract child models silently inherit ordering from non-abstract parents unless they explicitly set ordering in their own Meta. Setting ordering=None does NOT prevent inheritance — you must set ordering=[] to override with an empty list. `[llm-analyzed]`

## Notes from code
- [TODO] Handle multiple backends with different feature flags.

## Dependencies
- [Constraints](../constraints/_overview.md)
- [Django](../django/_overview.md)
- [Expressions](../expressions/_overview.md)
- [Fields](../fields/_overview.md)
- [Indexes](../indexes/_overview.md)
- [Lookups](../lookups/_overview.md)
- [Options](../options/_overview.md)
- [Proxy](../proxy/_overview.md)

## Referenced by
- [Article](../article/_overview.md)
- [Bar](../bar/_overview.md)
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
- [Indexes](../indexes/_overview.md)
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