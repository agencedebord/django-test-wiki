---
domain: views
confidence: llm-analyzed
last_updated: 2026-04-03
related_files:
  - tests/view_tests/views.py
  - tests/view_tests/models.py
  - tests/view_tests/urls.py
  - tests/view_tests/generic_urls.py
  - tests/view_tests/default_urls.py
  - tests/view_tests/tests/test_debug.py
  - tests/view_tests/tests/test_defaults.py
  - tests/view_tests/tests/test_csrf.py
  - tests/view_tests/tests/test_i18n.py
  - tests/view_tests/tests/test_json.py
  - tests/view_tests/tests/test_specials.py
  - tests/view_tests/tests/test_static.py
  - tests/middleware/views.py
  - tests/handlers/views.py
---

# Views

## What this domain does

This domain contains the test suite for Django's built-in view layer (`django.views`). It covers the default error handler views (400, 403, 404, 500), the debug/exception reporting system, CSRF failure views, internationalization views (JavaScript/JSON translation catalogs, language switching), static file serving, and JSON responses. It also provides test view functions used by other test domains (middleware, handlers) to exercise request/response handling behaviors. `[llm-analyzed]`

## Key behaviors

- **Default error views**: Tests `django.views.defaults` (page_not_found, server_error, bad_request, permission_denied) to verify correct HTTP status codes, template rendering, and content-type headers for error responses.
- **Debug and exception reporting**: Extensive tests for `django.views.debug` including `technical_500_response`, `technical_404_response`, `ExceptionReporter`, and `SafeExceptionReporterFilter`. Validates that sensitive variables and POST parameters are properly hidden in error reports via `@sensitive_variables()` and `@sensitive_post_parameters()` decorators. Includes async variants of all sensitive-data views.
- **Custom exception reporters**: Tests the ability to override `ExceptionReporter` via `request.exception_reporter_class` and `request.exception_reporter_filter` for per-request customization of error output.
- **CSRF failure view**: Tests `django.views.csrf.csrf_failure` for localized error messages and correct 403 responses when CSRF verification fails.
- **i18n views**: Tests `JavaScriptCatalog` and `JSONCatalog` for serving translation strings to JavaScript, including multi-package catalog merging and language switching via `set_language`.
- **Static file serving**: Tests `django.views.static.serve` for correct content delivery, Content-Length headers, Content-Encoding for gzip files, directory indexes, and `was_modified_since` conditional responses.
- **JSON responses**: Tests `JsonResponse` serialization of complex types (datetime, Decimal) through a dedicated view.
- **Non-ASCII redirect handling**: Validates that `RedirectView` properly handles non-ASCII URL targets (e.g., Chinese characters in paths).
- **Handler test views**: `tests/handlers/views.py` provides views for testing WSGI/ASGI handler behavior -- regular responses, streaming responses, no-response callables, transaction-awareness, and async views. Includes `CoroutineClearingView` to simulate the common async error of returning an unawaited coroutine.
- **Middleware test views**: `tests/middleware/views.py` provides views for CSP (Content Security Policy) testing -- nonce retrieval, policy overrides via `@csp_override`/`@csp_report_only_override`, CSP report collection, and the `@no_append_slash` decorator on both FBVs and CBVs.

## Domain interactions

- **Middleware domain**: `tests/middleware/views.py` defines views consumed by middleware tests to verify CSP headers, CSRF exemption, and URL slash-appending behavior.
- **Handlers domain**: `tests/handlers/views.py` defines views consumed by handler tests to verify WSGI/ASGI request processing, streaming, transactions, and async support.
- **URLs domain**: URL configuration files (`urls.py`, `generic_urls.py`, `default_urls.py`) wire views to URL patterns and are used by test cases that exercise routing.
- **Auth domain**: `generic_urls.py` references `django.contrib.auth.views` for login/logout URL patterns used in redirect tests.
- **Generic views domain**: `tests/generic_views/views.py` is a separate domain covering class-based generic views (DetailView, ListView, etc.), distinct from this domain which focuses on built-in utility views and error handlers.

## Gotchas and edge cases

- **CoroutineClearingView**: `tests/handlers/views.py` contains a view that deliberately returns an unawaited coroutine to test Django's detection of this common async mistake. It includes a `__del__` cleanup to suppress Python's "coroutine was never awaited" warning.
- **NoResponse callable**: A callable class in `tests/handlers/views.py` that returns `None` instead of an `HttpResponse`, used to test handler robustness.
- **Sensitive data filtering in async views**: The domain includes both sync and async variants of sensitive-data views, plus nested async call chains, to ensure `@sensitive_variables` works correctly across coroutine boundaries.
- **UnsafeExceptionReporterFilter**: A test-only filter that deliberately bypasses all security filtering, used to verify that the custom filter mechanism properly overrides the default safe behavior.
- **CSP report collector**: `tests/middleware/views.py` uses a module-level `csp_reports` list to collect CSP violation reports across requests -- this shared state could cause test pollution if tests are not properly isolated.
- **Template override for ExceptionReporter**: `TemplateOverrideExceptionReporter` demonstrates overriding `html_template_path` and `text_template_path` for custom 500 error pages.

## Dependencies

- [Base](../base/_overview.md)
- [Constraints](../constraints/_overview.md)
- [Django](../django/_overview.md)
- [Expressions](../expressions/_overview.md)
- [Indexes](../indexes/_overview.md)
- [Lookups](../lookups/_overview.md)
- [Options](../options/_overview.md)

## Referenced by

- [Urls](../urls/_overview.md) -- URL configurations reference view functions
- [Tests](../tests/_overview.md) -- various test modules import views for integration testing
- [Tests-custom-error-handlers](../tests-custom-error-handlers/_overview.md) -- custom error handler tests depend on view infrastructure
