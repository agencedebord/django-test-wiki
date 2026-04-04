---
domain: http
confidence: llm-analyzed
last_updated: 2026-04-03
related_files:
  - django/http/multipartparser.py
  - django/http/request.py
  - django/http/__init__.py
  - django/http/response.py
  - django/http/cookie.py
  - django/middleware/http.py
---

# Http

## What this domain does
The `http` domain is Django's entire HTTP abstraction layer: it models incoming requests, outgoing responses, header management, cookie handling, multipart file upload parsing, content negotiation, and conditional GET caching. It is one of the most widely imported domains in the codebase — nearly every other domain touches it. Its primary job is to translate raw WSGI/ASGI environ dicts and byte streams into typed Python objects, and to serialize Python objects back into compliant HTTP wire format. `[llm-analyzed]`

## Key behaviors
- **Multipart parsing defers to upload handlers before doing its own work**: MultiPartParser calls `handler.handle_raw_input()` on each upload handler before touching the stream itself. If any handler returns a non-None result, parsing stops immediately and that result is used. This is the extension point for custom upload processing (e.g., streaming directly to S3) — if a handler short-circuits here, MultiPartParser's own chunked boundary logic never runs. `[llm-analyzed]`
- **Chunk size is capped at 2^31-4 for 32-bit network API compatibility**: MultiPartParser computes chunk size as `min([2**31 - 4, *handler_chunk_sizes])`. The 2^31-4 cap exists because low-level network APIs use signed 32-bit integers for buffer sizes. The -4 ensures alignment. This is a portability constraint, not a performance tuning choice. `[llm-analyzed]`
- **Uploaded file resources are explicitly closed on parse exceptions**: MultiPartParser wraps its `_parse()` call and, on any exception, closes all open file objects in `self._files`. This is necessary because exceptions prevent normal GC-triggered cleanup, and open file handles would leak for the lifetime of the request cycle. Normal request completion relies on the Request object closing files instead. `[llm-analyzed]`
- **ConditionalGetMiddleware only fires on GET, never HEAD**: The middleware skips processing for any non-GET request. The comment explains HEAD specifically: the response body is always empty for HEAD, so computing an accurate ETag is impossible. For unsafe methods (POST, PUT, etc.), a 412 Precondition Failed response would be too late to be useful by the time process_response runs. `[llm-analyzed]`
- **ETag generation is suppressed when Cache-Control: no-store is set**: `needs_etag()` parses Cache-Control headers and returns False if any directive is `no-store`. This prevents adding ETag fingerprinting to responses explicitly marked as non-cacheable and non-storable — attaching an ETag to a `no-store` response would be semantically contradictory. `[llm-analyzed]`

## Domain interactions
- **Core**: Depends on core.exceptions for RequestDataTooBig, TooManyFieldsSent, TooManyFilesSent, SuspiciousMultipartForm, DisallowedHost, BadRequest, ImproperlyConfigured. These are raised during request parsing and host validation. Also uses core.files.uploadhandler for the SkipFile/StopUpload/StopFutureHandlers signals that control multipart parsing flow. `[llm-analyzed]`
- **Conf**: Reads settings.DEFAULT_CHARSET, settings.DATA_UPLOAD_MAX_NUMBER_FIELDS, settings.USE_X_FORWARDED_HOST, settings.ALLOWED_HOSTS, settings.DEBUG at parse/response-build time. These are not cached — they are read on each request, so runtime settings changes (e.g., in tests with override_settings) take effect immediately. `[llm-analyzed]`
- **Utils**: Relies heavily on utils: `parse_header_parameters` for Content-Type parsing, `iri_to_uri` and `escape_uri_path` for URL encoding, `is_same_domain` for host validation, `http_date` and `content_disposition_header` for response headers, `cc_delim_re` and `get_conditional_response` for ETag/caching logic, `CaseInsensitiveMapping` for header storage. `[llm-analyzed]`
- **Middleware**: ConditionalGetMiddleware lives in django/middleware/http.py but is architecturally part of the http domain's caching contract. It wraps any GET response to add ETag headers and convert responses to 304 Not Modified when appropriate, transparently to views. `[llm-analyzed]`
- **Django (asgiref)**: response.py imports asgiref's async_to_sync and sync_to_async, enabling HttpResponse subclasses to work in both ASGI and WSGI contexts. The http domain is the boundary where sync/async context switching happens for responses. `[llm-analyzed]`

## Gotchas and edge cases
- WSGIRequest does NOT call super().__init__(). Any variable initialized in HttpRequest.__init__() must be duplicated in WSGIRequest.__init__() or it will be missing on WSGI requests. This is an explicit maintenance trap noted in a code comment. `[llm-analyzed]`
- RawPostDataException is raised if you access request.body after request.POST or request.FILES has already been read from a multipart request. Reading from the parsed form data exhausts the input stream, making raw access impossible after the fact. `[llm-analyzed]`
- MultiPartParser boundary validation uses the regex `[ -~]{0,200}[!-~]` — only printable ASCII, max ~201 chars. Non-ASCII characters in the Content-Type boundary field raise MultiPartParserError, even though RFC 7578 technically allows more exotic boundaries. `[llm-analyzed]`
- The ResponseHeaders mapping stores keys lowercased internally but preserves the original casing in the stored tuple `(key, value)`. This means iteration yields the original casing, but lookup is case-insensitive. Do not rely on a specific casing when reading keys back out. `[llm-analyzed]`
- BadHeaderError inherits from ValueError, not from a custom HTTP exception base. Code catching ValueError for unrelated reasons may accidentally swallow header injection errors if they occur in the same scope. `[llm-analyzed]`

## Notes from code
- [FIXME] this currently assumes that upload handlers store the file as

## Dependencies
- [Base](../base/_overview.md)
- [Conf](../conf/_overview.md)
- [Core](../core/_overview.md)
- [Django](../django/_overview.md)
- [Utils](../utils/_overview.md)

## Referenced by
- [Base](../base/_overview.md)
- [Cond-get-urls](../cond-get-urls/_overview.md)
- [Conf](../conf/_overview.md)
- [Contrib](../contrib/_overview.md)
- [Core](../core/_overview.md)
- [Db](../db/_overview.md)
- [Django](../django/_overview.md)
- [Django-urls](../django-urls/_overview.md)
- [Django-views](../django-views/_overview.md)
- [Middleware](../middleware/_overview.md)
- [Templates](../templates/_overview.md)
- [Templatetags](../templatetags/_overview.md)
- [Test-security](../test-security/_overview.md)
- [Tests](../tests/_overview.md)
- [Utils](../utils/_overview.md)
- [Views](../views/_overview.md)