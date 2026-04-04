---
domain: templates
confidence: llm-analyzed
last_updated: 2026-04-03
related_files:
  - django/template/library.py
  - django/template/backends/django.py
  - django/template/backends/jinja2.py
  - django/template/backends/utils.py
  - django/template/backends/dummy.py
  - django/template/backends/base.py
  - django/template/__init__.py
  - django/template/response.py
  - django/template/smartif.py
  - django/template/context_processors.py
  - django/template/defaultfilters.py
  - django/template/engine.py
  - django/template/context.py
  - django/template/utils.py
  - django/template/loader.py
  - django/template/loader_tags.py
  - django/template/exceptions.py
  - django/template/autoreload.py
  - django/template/loaders/filesystem.py
  - django/template/loaders/cached.py
  - django/template/loaders/app_directories.py
  - django/template/loaders/base.py
  - django/template/loaders/locmem.py
  - django/template/base.py
  - django/template/defaulttags.py
---

# Templates

## What this domain does
The templates domain implements Django's template system, which has two distinct subsystems intentionally kept in one package: (1) a **Multiple Template Engines** abstraction layer (backends, EngineHandler, loader API) that allows pluggable backends like Jinja2, and (2) the **Django Template Language (DTL)** itself — a two-phase compile-then-render system. The domain solves template loading, compilation, context management, tag/filter registration, and rendering. The two subsystems were deliberately kept together (despite being conceptually separate) to make the Multiple Template Engines ... `[llm-analyzed]`

## Key behaviors
- **Two-phase template compilation: tokenize → parse → render**: Template compilation is split into Lexer (tokenize string into TEXT/VAR/BLOCK/COMMENT tokens) and Parser (convert tokens to a NodeList of Node objects). Rendering calls each Node's render() with a Context. The Template class wraps this pipeline. Compilation happens once at instantiation; render() can be called many times with different contexts. `[llm-analyzed]`
- **Debug mode switches the Lexer implementation**: Template.compile_nodelist() selects DebugLexer (vs Lexer) based on engine.debug. DebugLexer annotates exceptions with line/position info from the source. This is the only behavioral difference in debug mode at the compilation level. `[llm-analyzed]`
- **Context is a stack of dicts with built-in True/False/None**: BaseContext always initializes with a bottom-level dict containing {'True': True, 'False': False, 'None': None}. Variable lookup traverses from top of stack to bottom. This means templates can always use True/False/None without the caller explicitly passing them. push()/pop() are used with the 'with' statement via ContextDict.__enter__/__exit__. `[llm-analyzed]`
- **RenderContext is separate from variable Context and is template-scoped**: RenderContext stores template-local state (e.g. for cycle nodes, forloop variables) and is pushed/popped around each template render via push_state(). Crucially, RenderContext only resolves variables from the top of its own stack — variables don't bleed across included/extended templates. This is intentionally stricter scoping than the variable Context. `[llm-analyzed]`
- **Template.render() guards against double-binding but allows nested renders**: Template.render() calls context.render_context.push_state(self) unconditionally, then conditionally calls context.bind_template(self) only if context.template is None. If a template is already bound (i.e., during inheritance/inclusion), it skips bind_template and calls _render() directly. This allows {% extends %} and {% include %} to work without raising the 'already bound' RuntimeError. `[llm-analyzed]`

## Domain interactions
- **Django-views**: Views use SimpleTemplateResponse and TemplateResponse (from template.response) to defer rendering until after middleware has run. The response holds a template name and context; actual rendering happens in response.render() which calls template.render(). `[llm-analyzed]`
- **Conf**: EngineHandler reads settings.TEMPLATES lazily via cached_property. Engine reads settings indirectly through EngineHandler. Misconfigured TEMPLATES raises ImproperlyConfigured. `[llm-analyzed]`
- **Forms**: Forms use template rendering for their widget output. The template domain provides the rendering pipeline; forms provide context data. `[llm-analyzed]`
- **Core**: Uses django.core.exceptions.ImproperlyConfigured for misconfigured engine settings. Uses django.apps.apps.get_app_configs() in get_app_template_dirs() to discover template directories in installed apps. `[llm-analyzed]`
- **Utils**: Heavy dependency: uses SafeString/mark_safe for autoescaping, conditional_escape for filter output, smart_split for tokenizing template tags, localize/template_localtime for l10n, lazy_annotations for introspection of tag functions, cached_property for engine initialization. `[llm-analyzed]`
- **Templatetags**: templatetags domain consumes Library from this domain to register custom tags and filters. The Engine loads these via import_library() and makes them available to Parser instances. `[llm-analyzed]`
- **Contrib**: Admin, auth, and other contrib apps register their own template loaders, context processors, and template tag libraries. They depend on the engines singleton and Library registration APIs. `[llm-analyzed]`

## Gotchas and edge cases
- Template('...').render(Context({})) works without any explicit engine — but only if exactly one DjangoTemplates backend is configured. With zero or multiple backends, Engine.get_default() raises ImproperlyConfigured. This is a backwards-compat footgun for multi-engine setups. `[llm-analyzed]`
- context.bind_template() raises RuntimeError if called when a template is already bound. But Template.render() deliberately skips bind_template in that case. Code that calls bind_template directly (outside Template.render) must manage this invariant manually. `[llm-analyzed]`
- RenderContext variable lookup only checks the top of its stack (dicts[-1]), unlike BaseContext which searches all levels. This is intentional for template-local scoping but means you cannot read RenderContext values set in a parent template from an included template. `[llm-analyzed]`
- True, False, None are always available in templates as built-ins at the bottom of the context stack. If a caller passes {'True': 'something_else'} in their context dict, it will override these built-ins since the caller's dict is pushed on top. `[llm-analyzed]`
- get_app_template_dirs() is @lru_cache'd — it returns the same tuple for the lifetime of the process. Dynamic changes to installed apps after startup won't be reflected. `[llm-analyzed]`

## Dependencies
- [Base](../base/_overview.md)
- [Conf](../conf/_overview.md)
- [Constraints](../constraints/_overview.md)
- [Core](../core/_overview.md)
- [Db](../db/_overview.md)
- [Dispatch](../dispatch/_overview.md)
- [Django](../django/_overview.md)
- [Django-apps](../django-apps/_overview.md)
- [Django-urls](../django-urls/_overview.md)
- [Expressions](../expressions/_overview.md)
- [Forms](../forms/_overview.md)
- [Http](../http/_overview.md)
- [Indexes](../indexes/_overview.md)
- [Lookups](../lookups/_overview.md)
- [Middleware](../middleware/_overview.md)
- [Options](../options/_overview.md)
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
- [Good](../good/_overview.md)
- [Templatetags](../templatetags/_overview.md)
- [Tests](../tests/_overview.md)
- [Tests-custom-error-handlers](../tests-custom-error-handlers/_overview.md)
- [Utils](../utils/_overview.md)