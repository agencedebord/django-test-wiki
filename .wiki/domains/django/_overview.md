---
domain: django
confidence: llm-analyzed
last_updated: 2026-04-03
related_files:
  - django/test/signals.py
  - django/test/runner.py
  - django/test/client.py
  - django/test/html.py
  - django/test/__init__.py
  - django/test/selenium.py
  - django/test/utils.py
  - django/test/testcases.py
  - django/shortcuts.py
  - django/__init__.py
  - django/__main__.py
---

# Django

## What this domain does
The `django` domain is the framework root — it bootstraps the entire Django runtime and exposes cross-cutting convenience utilities. It is the foundation every other domain depends on, either directly (for shortcuts and setup) or transitively (as the framework itself). This is Django 6.1.0 (alpha). `[llm-analyzed]`

## Key behaviors
- **Bootstrap sequence via django.setup()**: django.setup() is the mandatory initialization entry point. It performs three things in strict order: (1) configure logging via settings.LOGGING_CONFIG/LOGGING, (2) set the URL script prefix from settings.FORCE_SCRIPT_NAME (defaults to '/' if None), (3) populate the app registry from settings.INSTALLED_APPS. The order matters — logging is configured before apps are populated, so app-level import errors are captured. set_prefix=False skips the URL prefix step, used in management commands and contexts where no HTTP request prefix applies. `[llm-analyzed]`
- **resolve_url() uses duck-typing priority chain**: resolve_url() tries resolution in this order: (1) if the target has get_absolute_url(), call it (model instances); (2) if it's a lazy string (Promise), materialize it first; (3) if it starts with './' or '../', return as-is (relative URLs); (4) try reverse() URL resolution by name; (5) if reverse() raises NoReverseMatch, re-raise if the target is callable OR if it doesn't look like a URL (no '/' or '.'). This last step is the silent fallthrough: bare URL strings like 'http://example.com' are returned as-is without validation. `[llm-analyzed]`
- **get_object_or_404 / get_list_or_404 accept Model, Manager, or QuerySet**: These shortcuts duck-type the first argument: if it has _default_manager, they call .all() on it (handles Model class and Manager); otherwise they use it directly (handles QuerySet). The distinction matters: passing a Manager lets callers use custom manager filters by default; passing a QuerySet lets callers chain filters before the 404 check. Passing anything else raises ValueError immediately rather than hitting the database. `[llm-analyzed]`
- **Async variants for all database-touching shortcuts**: aget_object_or_404(), aget_list_or_404() are async-native equivalents. They check for .aget() and .filter() respectively (not .get()/.filter() like the sync versions), meaning a QuerySet-like object must support the async ORM protocol. The async list variant uses an async comprehension over the filtered queryset rather than list() wrapping. `[llm-analyzed]`
- **redirect() supports permanent, temporary, and method-preserving redirects**: The preserve_request=True flag instructs the user agent to preserve the HTTP method and body (issues 307/308 instead of 302/301 semantically, though the actual class used is still HttpResponseRedirect/HttpResponsePermanentRedirect — the preserve_request flag is passed through to those response classes). This is a newer addition; older code only used permanent=True/False. `[llm-analyzed]`

## Domain interactions
- **Apps**: setup() calls apps.populate(settings.INSTALLED_APPS) to build the app registry. Every other domain that needs model discovery depends on this having been called first. `[llm-analyzed]`
- **Conf**: setup() accesses settings lazily (first access triggers configuration). FORCE_SCRIPT_NAME and LOGGING_CONFIG/LOGGING are read during bootstrap. `[llm-analyzed]`
- **Django-urls**: setup() calls set_script_prefix(); shortcuts.py imports reverse() and NoReverseMatch. resolve_url() is the single abstraction that turns model instances, view names, or raw URLs into a URL string. `[llm-analyzed]`
- **Http**: shortcuts.py imports HttpResponse, HttpResponseRedirect, HttpResponsePermanentRedirect, and Http404. The render() and redirect() shortcuts are thin wrappers that assemble these response objects. `[llm-analyzed]`
- **Templates**: render() delegates to loader.render_to_string(), which is the template layer entry point. The 'using' parameter selects a specific template engine by name. `[llm-analyzed]`
- **Utils**: Uses get_version() for __version__, configure_logging() during setup, and Promise/gettext for lazy string handling in shortcuts. `[llm-analyzed]`

## Gotchas and edge cases
- django.setup() must be called exactly once before any model or app registry access. Calling it multiple times is safe (apps.populate() is idempotent) but calling it zero times causes cryptic AppRegistryNotReady errors in any domain that imports models at module level. `[llm-analyzed]`
- resolve_url() silently passes through strings that look like URLs (contain '/' or '.') even if they are invalid. There is no URL format validation — garbage strings like 'not/a/real/url' will be returned as-is rather than raising an error. `[llm-analyzed]`
- get_object_or_404() raises MultipleObjectsReturned (not Http404) when multiple objects match. Only DoesNotExist is caught. Callers must handle MultipleObjectsReturned separately if their query is not guaranteed unique. `[llm-analyzed]`
- The VERSION tuple shows 6.1.0 alpha — this is pre-release Django. Any behavior documented here may differ from stable releases and should be verified against the final release notes. `[llm-analyzed]`
- python -m django (via __main__.py) is equivalent to django-admin but requires the package to be importable. It does NOT call django.setup() automatically — management commands handle their own setup. `[llm-analyzed]`

## Notes from code
- [TODO] Modify if/when that internal API is refactored

## Dependencies
- [Base](../base/_overview.md)
- [Conf](../conf/_overview.md)
- [Constraints](../constraints/_overview.md)
- [Contrib](../contrib/_overview.md)
- [Core](../core/_overview.md)
- [Db](../db/_overview.md)
- [Dispatch](../dispatch/_overview.md)
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
- [Apps](../apps/_overview.md)
- [Article](../article/_overview.md)
- [Bar](../bar/_overview.md)
- [Base](../base/_overview.md)
- [Cond-get-urls](../cond-get-urls/_overview.md)
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
- [Dispatch](../dispatch/_overview.md)
- [Django-apps](../django-apps/_overview.md)
- [Django-urls](../django-urls/_overview.md)
- [Django-views](../django-views/_overview.md)
- [Empty-join](../empty-join/_overview.md)
- [Extra-urls](../extra-urls/_overview.md)
- [Foo](../foo/_overview.md)
- [Forms](../forms/_overview.md)
- [Good](../good/_overview.md)
- [Http](../http/_overview.md)
- [Invalid-models](../invalid-models/_overview.md)
- [Is-active](../is-active/_overview.md)
- [Middleware](../middleware/_overview.md)
- [Minimal](../minimal/_overview.md)
- [Models](../models/_overview.md)
- [Multi-table](../multi-table/_overview.md)
- [Namespace-package-base](../namespace-package-base/_overview.md)
- [Natural](../natural/_overview.md)
- [No-password](../no-password/_overview.md)
- [One-config-app](../one-config-app/_overview.md)
- [Person](../person/_overview.md)
- [Proxy](../proxy/_overview.md)
- [Publication](../publication/_overview.md)
- [Query-performing-app](../query-performing-app/_overview.md)
- [Staticfiles-config](../staticfiles-config/_overview.md)
- [Tablespaces](../tablespaces/_overview.md)
- [Tasks](../tasks/_overview.md)
- [Templates](../templates/_overview.md)
- [Templatetags](../templatetags/_overview.md)
- [Tenant](../tenant/_overview.md)
- [Test-csp](../test-csp/_overview.md)
- [Test-exception](../test-exception/_overview.md)
- [Test-security](../test-security/_overview.md)
- [Tests](../tests/_overview.md)
- [Tests-custom-error-handlers](../tests-custom-error-handlers/_overview.md)
- [Two-configs-app](../two-configs-app/_overview.md)
- [Two-configs-one-default-app](../two-configs-one-default-app/_overview.md)
- [Two-default-configs-app](../two-default-configs-app/_overview.md)
- [Urls](../urls/_overview.md)
- [Utils](../utils/_overview.md)
- [Uuid-pk](../uuid-pk/_overview.md)
- [Views](../views/_overview.md)
- [With-custom-email-field](../with-custom-email-field/_overview.md)
- [With-foreign-key](../with-foreign-key/_overview.md)
- [With-integer-username](../with-integer-username/_overview.md)
- [With-last-login-attr](../with-last-login-attr/_overview.md)
- [With-many-to-many](../with-many-to-many/_overview.md)
- [With-unique-constraint](../with-unique-constraint/_overview.md)