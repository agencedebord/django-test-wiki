---
domain: article
confidence: llm-analyzed
last_updated: 2026-04-03
related_files:
  - tests/foreign_object/models/article.py
  - tests/foreign_object/tests.py
  - tests/foreign_object/test_forms.py
  - tests/model_package/models/article.py
  - tests/model_package/tests.py
---

# Article

## What this domain does

The article domain provides test models used in two distinct Django test suites:

1. **`tests/foreign_object`** -- Models that exercise Django's `ForeignObject` (non-public API) for multi-column and language-aware foreign key relationships. The core pattern is an `Article` with an `ActiveTranslationField` that resolves to the currently active locale's `ArticleTranslation` row, filtering by `lang` at query time via `get_extra_restriction()` and `get_extra_descriptor_filter()`.

2. **`tests/model_package`** -- A minimal `Article` model that verifies Django can discover models defined inside a Python sub-package (`models/article.py`) rather than a single `models.py` file. It tests M2M table creation and column naming for packaged models.

## Key behaviors

### Foreign object translation system (`tests/foreign_object`)

- **`ActiveTranslationField`**: A custom `ForeignObject` subclass that injects a `ColConstraint` for the current language (`get_language()`) into every query. This means `article.active_translation` transparently returns the translation matching `django.utils.translation.get_language()` without the caller specifying a language filter. `[confirmed]`
- **`ActiveTranslationFieldWithQ`**: Variant that returns a `Q` object from `get_extra_descriptor_filter()` instead of a plain dict, exercising a different code path in the descriptor machinery.
- **`ArticleTranslationDescriptor`**: Overrides `ForwardManyToOneDescriptor.__set__` to avoid writing to local fields when setting a translation, since the relationship is virtual (no actual FK column on `Article`). `[confirmed]`
- **`ColConstraint`**: A lightweight SQL fragment object with `as_sql()`, used inside `get_extra_restriction()` to append `alias.lang = %s` to JOIN conditions.
- **`ArticleTranslation`**: Standard model with `unique_together = ("article", "lang")`, holding `title`, `body`, and nullable `abstract`.
- **`ArticleTag` / `ArticleIdea`**: Related models that test `related_name` / `related_query_name` behavior on `ForeignKey` and `ManyToManyField` when combined with `ForeignObject` fields on the same model.
- **`NewsArticle`**: Proxy-like subclass of `Article` (multi-table inheritance) that validates `select_related("active_translation")` works through inheritance.
- **Form exclusion**: `ForeignObject` fields are intentionally excluded from `ModelForm` -- the `test_forms.py` suite confirms `active_translation` never appears in form output.

### Model package discovery (`tests/model_package`)

- Validates that `Article` defined in `models/article.py` gets a proper M2M intermediate table (`model_package_article_publications`) with correct column names.
- Confirms models can reference each other via dotted strings (`"model_package.Publication"`) across sub-package boundaries.

## Domain interactions

- **`person` domain** (`tests/foreign_object/models/person.py`): Shares the `foreign_object` test app. `Person`, `Country`, `Group`, `Membership`, and `Friendship` models test multi-column composite FK patterns. They are imported alongside article models in the same `__init__.py`.
- **`customers` domain** (`tests/foreign_object/models/customers.py`): Also part of the `foreign_object` test app, testing `ForeignObject` with `Address`, `Contact`, `Customer`.
- **`publication` domain** (`tests/model_package/models/publication.py`): The M2M target for `Article.publications` in the model-package test.
- **Django translation framework**: `ActiveTranslationField` depends on `django.utils.translation.get_language()` at query time, meaning the active locale (set via `translation.override()` or middleware) directly controls which translation row is fetched.

## Gotchas and edge cases

- **`requires_unique_target = False`**: `ActiveTranslationField` sets this because multiple `ArticleTranslation` rows exist per article (one per language). Without this override, Django's FK validation would reject the relationship.
- **`related_name = "+"`**: Both `active_translation` and `active_translation_q` suppress reverse relations to avoid clashes, since they point to the same target model.
- **Descriptor does not set local fields**: `ArticleTranslationDescriptor.__set__` intentionally skips the normal FK behavior of setting `_id` attributes on the instance, because there is no concrete FK column on `Article`. Misunderstanding this can lead to confusion when debugging assignment behavior.
- **Language-dependent queries**: Any queryset involving `active_translation` is implicitly scoped to the current thread's active language. Tests wrap calls in `translation.override("fi")` or `translation.override("en")` -- forgetting this context manager in test code will produce inconsistent results.
- **Nullable join**: `active_translation` is `null=True`, so articles without a translation in the active language return `None` rather than raising.

## Dependencies

- [Base](../base/_overview.md) -- Django model base classes
- [Django](../django/_overview.md) -- `ForeignObject`, `ForwardManyToOneDescriptor`, translation utilities
- [Expressions](../expressions/_overview.md) -- `Q` objects used in `ActiveTranslationFieldWithQ`
- [Lookups](../lookups/_overview.md) -- query filtering on foreign object fields
- [Options](../options/_overview.md) -- `Meta.unique_together` on `ArticleTranslation`

## Referenced by

- `tests/foreign_object/tests.py` -- `MultiColumnFKTests` and translation-related test methods
- `tests/foreign_object/test_forms.py` -- `FormsTests` validating ForeignObject exclusion from ModelForm
- `tests/model_package/tests.py` -- `ModelPackageTests` validating sub-package model discovery
