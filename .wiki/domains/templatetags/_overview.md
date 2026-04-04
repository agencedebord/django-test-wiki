---
domain: templatetags
confidence: llm-analyzed
last_updated: 2026-04-03
related_files:
  - django/templatetags/l10n.py
  - django/templatetags/tz.py
  - django/templatetags/cache.py
  - django/templatetags/static.py
  - django/templatetags/i18n.py
---

# Templatetags

## What this domain does
The `templatetags` domain provides Django template filters and tags for four cross-cutting concerns: timezone conversion (`tz.py`), localization formatting (`l10n.py`), internationalization/translation (`i18n.py`), template fragment caching (`cache.py`), and static/media URL generation (`static.py`). Each module registers its tags and filters into a `Library` instance, making them available via `{% load %}` in templates. `[llm-analyzed]`

## Key behaviors
- **Timezone conversion via filters and block tags**: tz.py provides three layers: `localtime` filter (converts to active TZ), `utc` filter (converts to UTC), and `timezone` filter (converts to an arbitrary TZ). Block tags `{% localtime on/off %}` and `{% timezone 'Zone' %}` temporarily override `context.use_tz` and the active timezone for their content. The `{% get_current_timezone as VAR %}` tag stores the current TZ name in the context without rendering anything. `[llm-analyzed]`
- **datetimeobject subclass prevents auto-conversion loop**: The `do_timezone` filter returns a `datetimeobject` (a `datetime` subclass) with `convert_to_local_time = False` set as an instance attribute. This flag tells Django's template engine not to re-convert the already-converted value to local time during rendering. This is intentional design: plain `datetime` instances can't have arbitrary attributes assigned, hence the subclass hack. `[llm-analyzed]`
- **Naive datetime handling in timezone filter**: When `do_timezone` receives a naive datetime, it assumes it is in the default timezone (`settings.TIME_ZONE`) and makes it aware before converting. Any exception during this step causes the filter to silently return an empty string — filters must never raise exceptions per Django convention. `[llm-analyzed]`
- **Localization block tags toggle context.use_l10n**: The `{% localize on/off %}` block tag in l10n.py mutates `context.use_l10n` for the duration of its block, then restores the previous value. This allows overriding `USE_L10N` per-block in templates. The `localize` and `unlocalize` filters force localization state on individual values without affecting the block context. `[llm-analyzed]`
- **Translation tags handle plural forms, context, and variable interpolation**: i18n.py's `{% blocktranslate %}` converts `{{ var }}` tokens inside the block to `%(var)s` format strings, then passes them to gettext/ngettext. If the translated string cannot be formatted with the provided variables (KeyError/ValueError), it retries with `translation.override(None)` (i.e., the source language) as a fallback to avoid crashing. Percent signs in literal template text are doubled (`%%`) to prevent misinterpretation as format flags. `[llm-analyzed]`

## Domain interactions
- **Django (template engine)**: All tag/filter modules register with `template.Library()` and return `Node` subclasses. They depend on the template engine's context mutation API (`context.use_tz`, `context.use_l10n`, `context.update/pop`) and rendering pipeline conventions (filters must never raise, render returns strings). `[llm-analyzed]`
- **Utils (timezone, formats, translation)**: tz.py uses `django.utils.timezone` for aware/naive detection, conversion, and override context manager. l10n.py uses `django.utils.formats.localize`. i18n.py uses `django.utils.translation` for gettext, ngettext, pgettext, npgettext, and language info lookups. `[llm-analyzed]`
- **Core (cache)**: cache.py depends on `django.core.cache.caches` for backend lookup and `django.core.cache.utils.make_template_fragment_key` for cache key generation. The cache tag is a thin orchestration layer over these core utilities. `[llm-analyzed]`
- **Contrib (staticfiles)**: static.py conditionally imports `staticfiles_storage` from `django.contrib.staticfiles` at render time. The tag degrades gracefully when the app is absent, making staticfiles an optional dependency detected at runtime rather than import time. `[llm-analyzed]`
- **Conf (settings)**: i18n.py reads `settings.LANGUAGES` to populate available languages. static.py reads `settings.STATIC_URL` and `settings.MEDIA_URL` via `PrefixNode.handle_simple`. tz.py uses the default timezone from settings via `timezone.get_default_timezone()`. `[llm-analyzed]`

## Gotchas and edge cases
- The `datetimeobject` subclass and its `convert_to_local_time = False` attribute is a HACK explicitly noted in the code. Any code inspecting datetime types needs to account for this subclass, and the flag must be preserved if the object is transformed further. `[llm-analyzed]`
- The fragment name in `{% cache %}` cannot be a variable — it is taken as a literal token string. Using a variable name there will not resolve it; the variable name itself becomes the cache key fragment. `[llm-analyzed]`
- The `{% localize %}` and `{% localtime %}` block tags restore context state after their block using a save/restore pattern (not a context manager), so exceptions within the block will leave the context in a modified state. `[llm-analyzed]`
- In `BlockTranslateNode.render`, if the translated string format fails, the fallback re-renders in the source language using `translation.override(None)`. This means a malformed translation silently degrades to the original string, not an error. `[llm-analyzed]`
- Static URL resolution behavior changes at runtime based on whether `django.contrib.staticfiles` is in `INSTALLED_APPS`. The same template will produce different URLs in environments with and without that app. `[llm-analyzed]`

## Notes from code
- [HACK] datetime instances cannot be assigned new attributes. Define a subclass
- [HACK] the convert_to_local_time flag will prevent

## Dependencies
- [Base](../base/_overview.md)
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
- [Utils](../utils/_overview.md)

## Referenced by
- [Base](../base/_overview.md)
- [Conf](../conf/_overview.md)
- [Contrib](../contrib/_overview.md)
- [Core](../core/_overview.md)
- [Db](../db/_overview.md)
- [Django](../django/_overview.md)
- [Django-views](../django-views/_overview.md)
- [Forms](../forms/_overview.md)
- [Tests](../tests/_overview.md)
- [Utils](../utils/_overview.md)