---
domain: middleware
confidence: llm-analyzed
last_updated: 2026-04-03
related_files:
  - django/middleware/clickjacking.py
  - django/middleware/gzip.py
  - django/middleware/csrf.py
  - django/middleware/security.py
  - django/middleware/cache.py
  - django/middleware/csp.py
  - django/middleware/common.py
  - django/middleware/http.py
  - django/middleware/locale.py
---

# Middleware

## What this domain does
Django's middleware layer is a pipeline of request/response hooks that handles cross-cutting concerns: security hardening, caching, compression, CSRF protection, CSP enforcement, locale detection, URL normalization, and clickjacking prevention. Each middleware operates independently on the request/response cycle via Django's MiddlewareMixin, which supports both sync and async execution. Middleware is ordered in MIDDLEWARE settings, and order matters critically — especially for the cache pair and CSRF. `[llm-analyzed]`

## Key behaviors
- **CSRF uses masked tokens to prevent BREACH-style attacks**: The CSRF secret (32 chars) is never sent directly. Instead, get_token() always returns a freshly masked version: a random mask XOR-ed with the secret produces a 64-char token. On validation, _does_token_match() unmasks the submitted token back to the secret and compares with constant_time_compare. This means every page load gets a unique CSRF token in the HTML, defeating compression side-channel attacks (BREACH). The secret itself lives in either the cookie or session (CSRF_SESSION_KEY = '_csrftoken'). `[llm-analyzed]`
- **Cache middleware must be split and placed at opposite ends of MIDDLEWARE**: UpdateCacheMiddleware must be FIRST in MIDDLEWARE (runs last on response), FetchFromCacheMiddleware must be LAST (runs last on request). This is counterintuitive. If LocaleMiddleware is present, the two-part form is required — not the single CacheMiddleware — because locale affects cache keys. The cache is only populated when request._cache_update_cache is True, a flag set by FetchFromCacheMiddleware on miss. UpdateCacheMiddleware checks this flag before storing. `[llm-analyzed]`
- **Cache middleware refuses to store user-specific cookies set on cookie-less requests**: UpdateCacheMiddleware skips caching if: the request had no cookies, the response sets cookies, AND the response has Vary: Cookie. This prevents accidentally caching a response that sets a session/auth cookie for an anonymous request, which would leak that cookie to subsequent users. `[llm-analyzed]`
- **GZip only compresses if it actually reduces size, and weakens strong ETags**: For non-streaming responses, GZipMiddleware computes the compressed content and only replaces the response body if the compressed version is strictly shorter. For streaming responses, it has no choice but to compress blindly and must delete Content-Length. Importantly, any strong ETag (starts with '"') is converted to a weak ETag (W/"...") because compression changes the body, making the original strong ETag invalid per RFC 9110. `[llm-analyzed]`
- **CSP headers respect per-response overrides and never overwrite already-set headers**: ContentSecurityPolicyMiddleware checks response._csp_config and response._csp_ro_config before falling back to settings.SECURE_CSP and SECURE_CSP_REPORT_ONLY. Views can override CSP per-response by setting these attributes. Additionally, if the header is already present in the response (e.g., set manually in the view), the middleware never overwrites it. An empty config means no CSP header is added. `[llm-analyzed]`

## Domain interactions
- **Conf**: Every middleware reads from django.conf.settings. SecurityMiddleware is unique in caching all security settings at __init__ time rather than per-request. CSP middleware falls back to settings.SECURE_CSP / SECURE_CSP_REPORT_ONLY when the response doesn't override them. `[llm-analyzed]`
- **Utils**: Heavy dependency. Uses django.utils.cache (patch_vary_headers, get_cache_key, learn_cache_key, get_conditional_response, patch_response_headers), django.utils.crypto (constant_time_compare, get_random_string), django.utils.http (parse_http_date_safe, is_same_domain, escape_leading_slashes), django.utils.csp (CSP, LazyNonce, build_policy), django.utils.deprecation (MiddlewareMixin), django.utils.functional (cached_property). `[llm-analyzed]`
- **Core**: Uses django.core.cache (caches, DEFAULT_CACHE_ALIAS) for the cache middleware backend. Uses django.core.exceptions (PermissionDenied, ImproperlyConfigured, DisallowedHost) for error signaling. Uses django.core.mail (mail_managers) for broken link notifications. `[llm-analyzed]`
- **Django-urls**: CommonMiddleware and LocaleMiddleware use is_valid_path() to check if a slash-appended or language-prefixed URL resolves. CsrfViewMiddleware uses get_callable() to load CSRF_FAILURE_VIEW. LocaleMiddleware uses get_script_prefix() to correctly insert the language code after the WSGI script prefix. `[llm-analyzed]`
- **Http**: Middleware produces and inspects HttpResponse objects. CsrfViewMiddleware reads HttpHeaders and handles UnreadablePostError (client disconnect during POST body read). CommonMiddleware returns HttpResponsePermanentRedirect for APPEND_SLASH and PREPEND_WWW. LocaleMiddleware returns HttpResponseRedirect for language prefix redirects. `[llm-analyzed]`
- **Templates**: The CSP nonce (via get_nonce()) and CSRF token (via get_token()) are consumed by template context processors, making them available in templates as {{ request.csp_nonce }} and {% csrf_token %}. `[llm-analyzed]`
- **Views**: Provides the request._csp_nonce attribute consumed by views and templates. CSRF middleware exposes get_token(request) and rotate_token(request) as the public API for views to interact with CSRF tokens — rotate_token() should be called on login. `[llm-analyzed]`
- **Test-security**: SecurityMiddleware is directly tested here, including HSTS header generation, X-Content-Type-Options, SSL redirect logic, and referrer policy. Tests override the settings that SecurityMiddleware caches at __init__ time, requiring middleware re-instantiation. `[llm-analyzed]`

## Gotchas and edge cases
- Cache middleware order is reversed from intuition: UpdateCacheMiddleware must be FIRST in MIDDLEWARE (not last), FetchFromCacheMiddleware must be LAST. Getting this wrong means caching never works or responses are never stored. `[llm-analyzed]`
- SecurityMiddleware reads SECURE_* settings once at __init__, not per-request. In tests that override these settings via @override_settings, the middleware instance must be re-created or the override won't take effect. `[llm-analyzed]`
- CSRF tokens are never safe to store or reuse across requests — get_token() always returns a freshly masked token even for the same secret. Comparing two tokens from get_token() directly will always fail. `[llm-analyzed]`
- rotate_token() must be called on user login to prevent session fixation of the CSRF secret. Forgetting this is a security vulnerability. `[llm-analyzed]`
- GZipMiddleware converts strong ETags to weak ETags. If downstream code depends on strong ETag semantics (e.g., for byte-range requests), placing GZipMiddleware in the stack will silently break this. `[llm-analyzed]`

## Dependencies
- [Conf](../conf/_overview.md)
- [Core](../core/_overview.md)
- [Django](../django/_overview.md)
- [Django-urls](../django-urls/_overview.md)
- [Http](../http/_overview.md)
- [Utils](../utils/_overview.md)

## Referenced by
- [Base](../base/_overview.md)
- [Conf](../conf/_overview.md)
- [Contrib](../contrib/_overview.md)
- [Core](../core/_overview.md)
- [Db](../db/_overview.md)
- [Django](../django/_overview.md)
- [Django-views](../django-views/_overview.md)
- [Templates](../templates/_overview.md)
- [Templatetags](../templatetags/_overview.md)
- [Test-security](../test-security/_overview.md)
- [Tests](../tests/_overview.md)
- [Utils](../utils/_overview.md)
- [Views](../views/_overview.md)