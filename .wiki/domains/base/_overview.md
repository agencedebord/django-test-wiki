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
The "base" domain provides two foundational pillars of Django's runtime: the HTTP request handler chain (BaseHandler) and the ORM model class system (ModelBase + Model). It is the lowest-level layer that every other domain builds on — defining how requests are processed and how Python classes become database-mapped models. `[llm-analyzed]`

## Key behaviors
- **Middleware chain assembly (sync/async dual-mode)**: BaseHandler.load_middleware() builds the middleware stack in reverse order from settings.MIDDLEWARE, wrapping each layer with convert_exception_to_response. Crucially, it negotiates sync/async compatibility per middleware: if the handler is async but the middleware only supports sync, it adapts using async_to_sync; conversely, sync handlers adapt async middleware via sync_to_async. The _middleware_chain attribute is only set after full initialization, serving as an atomic 'ready' flag — partial initialization is never visible to callers. `[llm-analyzed]`
- **Exception middleware is forced synchronous**: process_exception hooks are always adapted to synchronous mode regardless of whether the overall handler is async. This is an intentional design constraint noted in the code: the exception-handling stack remains synchronous for now. Async middleware that defines process_exception will be wrapped with adapt_method_mode(False, ...), which applies async_to_sync. `[llm-analyzed]`
- **MiddlewareNotUsed short-circuits handler adaptation**: When a middleware raises MiddlewareNotUsed during initialization, the handler reverts to adapted_handler (the version before the failed middleware was applied), not to the version before adaptation. This means the adaptation cost for that layer is thrown away but the chain continues cleanly. In DEBUG mode, the reason string is logged. `[llm-analyzed]`
- **ModelBase metaclass bootstraps every Model subclass**: ModelBase.__new__ intercepts class creation for all Model subclasses. It separates attrs into contributable (those with contribute_to_class) and plain attrs, passes plain attrs to type.__new__ first to ensure __set_name__ is called correctly, then calls contribute_to_class for the rest. This ordering is critical: Django fields (descriptors) must be registered via contribute_to_class after the class exists, but simple attributes must be present during class body execution. `[llm-analyzed]`
- **Per-model exception classes are dynamically generated**: ModelBase creates DoesNotExist, MultipleObjectsReturned, and NotUpdated as subclasses of the corresponding exceptions from all non-abstract parent models in the MRO. This allows catching MyModel.DoesNotExist specifically while still being caught by the generic ObjectDoesNotExist. NotUpdated additionally inherits from DatabaseError for backward compatibility — __subclasshook__ is not used in exception handling, so explicit inheritance is required. `[llm-analyzed]`

## Domain interactions
- **Middleware**: Consumes settings.MIDDLEWARE to build the handler chain. Calls each middleware's __init__ with an adapted handler, then introspects for process_view, process_template_response, and process_exception hooks. Raises ImproperlyConfigured if a middleware factory returns None. `[llm-analyzed]`
- **Options**: ModelBase calls add_to_class('_meta', Options(meta, app_label)) on every new Model subclass. Options is the _meta object that stores all field definitions, ordering, db_table, and other model configuration. Everything downstream that reads model metadata goes through _meta. `[llm-analyzed]`
- **Fields**: Fields implement contribute_to_class, which ModelBase uses as the signal to defer their registration until after the class shell exists. Plain field attributes without contribute_to_class would be added directly by type.__new__ and bypass Django's field registration machinery. `[llm-analyzed]`
- **Dispatch**: ModelBase fires class_prepared signal after a model class is fully configured. Model.__init__ fires pre_init and post_init signals. Model.save fires pre_save and post_save. These signals are the primary extension points for third-party apps to hook into model lifecycle. `[llm-analyzed]`
- **Django-apps**: ModelBase calls apps.get_containing_app_config(module) to auto-detect which installed app a model belongs to, deriving app_label automatically. If the module is not under any installed app and no explicit app_label is set, concrete models raise RuntimeError at class creation time — not at import time of the app. `[llm-analyzed]`
- **Conf**: BaseHandler reads settings.MIDDLEWARE for the middleware list and settings.ROOT_URLCONF for URL resolution on each request. settings.DEBUG controls whether adaptation logging and MiddlewareNotUsed messages are emitted. `[llm-analyzed]`
- **Db**: Model.NotUpdated inherits from DatabaseError for backward compatibility with code that catches database errors broadly. The base module imports transaction, connections, and router for save/delete operations on model instances. `[llm-analyzed]`

## Gotchas and edge cases
- Middleware adaptation is negotiated lazily per-layer: a sync Django setup will wrap async-capable middleware with async_to_sync only if it has no sync_capable=True. A middleware that sets both sync_capable and async_capable may run in either mode depending on handler context. `[llm-analyzed]`
- The _middleware_chain assignment is intentionally last in load_middleware(). Any exception during middleware instantiation leaves _middleware_chain as None, which will cause a hard AttributeError on the first request — not a graceful startup failure. `[llm-analyzed]`
- process_exception is always sync-wrapped even in an async handler. Writing async-native exception handling in middleware will not work as expected; the method will be run synchronously via async_to_sync. `[llm-analyzed]`
- ModelBase skips initialization entirely for the Model class itself (not subclasses): `if not parents: return super_new(...)`. This means Model's own attrs bypass Django's metaclass machinery — only subclasses go through field registration. `[llm-analyzed]`
- contributable_attrs are separated from new_attrs before type.__new__ is called, meaning Django descriptor fields are NOT present in the class dict when __set_name__ runs for other attrs. Code relying on field presence during sibling attribute __set_name__ will fail silently. `[llm-analyzed]`

## Notes from code
- [TODO] Handle multiple backends with different feature flags.

## Dependencies
- [Conf](../conf/_overview.md)
- [Constraints](../constraints/_overview.md)
- [Contrib](../contrib/_overview.md)
- [Core](../core/_overview.md)
- [Db](../db/_overview.md)
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
- [Http](../http/_overview.md)
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