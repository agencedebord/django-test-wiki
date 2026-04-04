---
domain: core
confidence: llm-analyzed
last_updated: 2026-04-03
related_files:
  - django/core/signing.py
  - django/core/signals.py
  - django/core/cache/backends/memcached.py
  - django/core/cache/backends/db.py
  - django/core/cache/backends/filebased.py
  - django/core/cache/backends/redis.py
  - django/core/cache/backends/dummy.py
  - django/core/cache/backends/base.py
  - django/core/cache/backends/locmem.py
  - django/core/cache/__init__.py
  - django/core/cache/utils.py
  - django/core/mail/backends/console.py
  - django/core/mail/backends/filebased.py
  - django/core/mail/backends/smtp.py
  - django/core/mail/backends/dummy.py
  - django/core/mail/backends/base.py
  - django/core/mail/backends/locmem.py
  - django/core/mail/__init__.py
  - django/core/mail/message.py
  - django/core/mail/utils.py
  - django/core/checks/compatibility/django_4_0.py
  - django/core/checks/files.py
  - django/core/checks/caches.py
  - django/core/checks/security/sessions.py
  - django/core/checks/security/csrf.py
  - django/core/checks/security/base.py
  - django/core/checks/registry.py
  - django/core/checks/model_checks.py
  - django/core/checks/database.py
  - django/core/checks/__init__.py
  - django/core/checks/templates.py
  - django/core/checks/translation.py
  - django/core/checks/async_checks.py
  - django/core/checks/messages.py
  - django/core/checks/urls.py
  - django/core/checks/commands.py
  - django/core/validators.py
  - django/core/asgi.py
  - django/core/management/color.py
  - django/core/management/__init__.py
  - django/core/management/templates.py
  - django/core/management/utils.py
  - django/core/management/commands/createcachetable.py
  - django/core/management/commands/inspectdb.py
  - django/core/management/commands/squashmigrations.py
  - django/core/management/commands/check.py
  - django/core/management/commands/startapp.py
  - django/core/management/commands/optimizemigration.py
  - django/core/management/commands/sqlmigrate.py
  - django/core/management/commands/makemigrations.py
  - django/core/management/commands/sqlflush.py
  - django/core/management/commands/makemessages.py
  - django/core/management/commands/shell.py
  - django/core/management/commands/dumpdata.py
  - django/core/management/commands/test.py
  - django/core/management/commands/flush.py
  - django/core/management/commands/loaddata.py
  - django/core/management/commands/runserver.py
  - django/core/management/commands/showmigrations.py
  - django/core/management/commands/sqlsequencereset.py
  - django/core/management/commands/dbshell.py
  - django/core/management/commands/sendtestemail.py
  - django/core/management/commands/startproject.py
  - django/core/management/commands/migrate.py
  - django/core/management/commands/compilemessages.py
  - django/core/management/commands/diffsettings.py
  - django/core/management/commands/testserver.py
  - django/core/management/sql.py
  - django/core/management/base.py
  - django/core/serializers/pyyaml.py
  - django/core/serializers/__init__.py
  - django/core/serializers/xml_serializer.py
  - django/core/serializers/jsonl.py
  - django/core/serializers/python.py
  - django/core/serializers/json.py
  - django/core/serializers/base.py
  - django/core/paginator.py
  - django/core/files/locks.py
  - django/core/files/storage/mixins.py
  - django/core/files/storage/handler.py
  - django/core/files/storage/memory.py
  - django/core/files/storage/filesystem.py
  - django/core/files/storage/__init__.py
  - django/core/files/storage/base.py
  - django/core/files/uploadhandler.py
  - django/core/files/utils.py
  - django/core/files/uploadedfile.py
  - django/core/files/temp.py
  - django/core/files/images.py
  - django/core/files/move.py
  - django/core/files/base.py
  - django/core/exceptions.py
  - django/core/handlers/exception.py
  - django/core/handlers/asgi.py
  - django/core/handlers/base.py
  - django/core/handlers/wsgi.py
  - django/core/servers/basehttp.py
  - django/core/wsgi.py
---

# Core

## What this domain does
The `core` domain is Django's central infrastructure layer. It provides the complete request/response pipeline (both WSGI and ASGI), the middleware loading and adaptation system, exception-to-HTTP-response conversion, file upload handling, storage backend management, and the email sending API. It sits between the HTTP server and the rest of Django, translating raw server events into typed request objects and typed response objects back into server events. `[llm-analyzed]`

## Key behaviors
- **Middleware chain is built once at startup, not per-request**: BaseHandler.load_middleware() iterates settings.MIDDLEWARE in reverse and wraps each middleware around the previous one, building a single callable chain stored in _middleware_chain. This attribute doubles as an initialization flag — it is only assigned when the full chain is successfully constructed. If middleware raises MiddlewareNotUsed, it is silently skipped (with DEBUG logging). If a middleware factory returns None, ImproperlyConfigured is raised immediately. `[llm-analyzed]`
- **Exception middleware is always synchronous, even in async handlers**: In load_middleware(), process_exception hooks are explicitly adapted with adapt_method_mode(False, ...) — forcing sync mode regardless of whether the handler is async. This is an intentional design constraint: the exception-handling stack does not support async process_exception methods. Code adding new exception middleware must remain synchronous. `[llm-analyzed]`
- **Sync/async adaptation is automatic and bidirectional**: adapt_method_mode() uses asgiref's sync_to_async (thread_sensitive=True) to wrap sync callables for async contexts, and async_to_sync to wrap async callables for sync contexts. This means a sync-only middleware in an async handler gets wrapped automatically, but incurs a thread context switch per call. The DEBUG flag enables logging of these adaptations. `[llm-analyzed]`
- **ASGI handler only handles HTTP scope type — WebSocket and other types are rejected**: ASGIHandler.__call__ raises ValueError for any scope type other than 'http'. There is a FIXME comment in the code acknowledging this should be overridable but is not yet. This means Django's built-in ASGI handler cannot serve WebSocket connections natively — that requires a separate ASGI application (e.g., Django Channels). `[llm-analyzed]`
- **ASGI body reading has a 60-second timeout and aborts silently on RequestAborted**: ASGIHandler.handle() calls read_body(receive) which will raise RequestAborted if the client disconnects or the body_receive_timeout (60s) is exceeded. When RequestAborted is caught, the handler returns without sending any response. This is intentional — the client is gone, so there is nothing to respond to. `[llm-analyzed]`

## Domain interactions
- **Middleware**: BaseHandler.load_middleware() reads settings.MIDDLEWARE and constructs the middleware chain. It calls adapt_method_mode() to bridge sync/async boundaries between middleware and handler. Each middleware's process_view, process_template_response, and process_exception hooks are extracted and stored in separate lists on the handler. `[llm-analyzed]`
- **Http**: Both WSGIRequest and ASGIRequest subclass HttpRequest. The handlers parse raw server data (environ dict or ASGI scope) into HttpRequest fields (META, path, method, _stream). They also use QueryDict, parse_cookie, FileResponse, and HttpResponseServerError from http. `[llm-analyzed]`
- **Django-urls**: BaseHandler calls set_urlconf(settings.ROOT_URLCONF) on each request to initialize the URL resolver for that thread. resolve_request() and get_resolver() are used to dispatch to views and to resolve error handlers (404, 403, 400, 500) from the URL configuration. `[llm-analyzed]`
- **Dispatch**: The exception handler fires signals: request_started (before processing), got_request_exception (for uncaught exceptions and error handler fallbacks). The ASGI handler sends request_started as an async signal. `[llm-analyzed]`
- **Conf**: Handlers read settings.MIDDLEWARE, settings.DEBUG, settings.DEBUG_PROPAGATE_EXCEPTIONS, settings.FORCE_SCRIPT_NAME, settings.ROOT_URLCONF, settings.STORAGES, and settings.EMAIL_BACKEND. These are all read at startup or per-request, never cached at import time. `[llm-analyzed]`
- **Db**: BaseHandler._get_response calls make_view_atomic() which may wrap the view in a database transaction if the view or its URL pattern specifies atomic behavior. After the response, connections.close_old_connections() is typically called via the request_finished signal (wired elsewhere). `[llm-analyzed]`
- **Django-views**: The exception handler delegates to debug.technical_404_response, debug.technical_500_response, and resolver.resolve_error_handler() to produce error responses. It force-renders TemplateResponse objects returned by error views. `[llm-analyzed]`
- **Forms**: File upload handlers in core.files.uploadhandler produce UploadedFile objects (InMemoryUploadedFile, TemporaryUploadedFile) that are consumed by the multipart form parser and ultimately surfaced in request.FILES. `[llm-analyzed]`

## Gotchas and edge cases
- The ASGI handler currently cannot serve WebSocket connections — there is a FIXME in the code acknowledging this limitation. Any non-HTTP scope type raises ValueError. `[llm-analyzed]`
- Exception middleware (process_exception) is always synchronous. Even in a fully async handler setup, these hooks are force-adapted to sync mode. Adding an async process_exception method will not work as expected. `[llm-analyzed]`
- In DEBUG mode, BadRequest exceptions are rendered using the 500 debug template but served with a 400 status code. The response looks like a 500 error page in the browser dev tools if you're not checking the status line. `[llm-analyzed]`
- SecurityOperation subclasses (DisallowedHost, etc.) log to django.security.* loggers, not django.request. If you configure logging only on django.request, security events will be silently dropped. `[llm-analyzed]`
- RequestDataTooBig, TooManyFieldsSent, and TooManyFilesSent mark the request POST as unreadable (_mark_post_parse_error). Any middleware or code that tries to access request.POST after the exception handler runs will get an empty result or re-raise, not the original data. `[llm-analyzed]`

## Notes from code
- [FIXME] Allow to override this.

## Dependencies
- [Base](../base/_overview.md)
- [Conf](../conf/_overview.md)
- [Constraints](../constraints/_overview.md)
- [Contrib](../contrib/_overview.md)
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
- [Base](../base/_overview.md)
- [Conf](../conf/_overview.md)
- [Constraints](../constraints/_overview.md)
- [Contrib](../contrib/_overview.md)
- [Db](../db/_overview.md)
- [Django](../django/_overview.md)
- [Django-apps](../django-apps/_overview.md)
- [Django-urls](../django-urls/_overview.md)
- [Django-views](../django-views/_overview.md)
- [Forms](../forms/_overview.md)
- [Http](../http/_overview.md)
- [Middleware](../middleware/_overview.md)
- [Tasks](../tasks/_overview.md)
- [Templates](../templates/_overview.md)
- [Templatetags](../templatetags/_overview.md)
- [Test-exception](../test-exception/_overview.md)
- [Tests](../tests/_overview.md)
- [Tests-custom-error-handlers](../tests-custom-error-handlers/_overview.md)
- [Utils](../utils/_overview.md)
- [Views](../views/_overview.md)