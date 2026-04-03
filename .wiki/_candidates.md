# Memory Candidates

> Auto-generated proposals to confirm, reject, or reformulate.
> These candidates are not yet confirmed memory.
> Edit this file or use `codefidence promote <id>` to validate.

## django

### django-001

- **status**: pending
- **type**: exception
- **confidence**: inferred
- **provenance**:
  - file: django/core/management/commands/squashmigrations.py
- **rationale**: File path 'django/core/management/commands/squashmigrations.py' contains exception naming pattern (legacy/compat/override)
- **target**: .wiki/domains/django/_overview.md

> squashmigrations seems to contain a special case or legacy behavior

**Action** : `confirm` | `reformulate` | `reject`

### django-002

- **status**: pending
- **type**: exception
- **confidence**: inferred
- **provenance**:
  - file: django/core/management/commands/optimizemigration.py
- **rationale**: File path 'django/core/management/commands/optimizemigration.py' contains exception naming pattern (legacy/compat/override)
- **target**: .wiki/domains/django/_overview.md

> optimizemigration seems to contain a special case or legacy behavior

**Action** : `confirm` | `reformulate` | `reject`

### django-003

- **status**: pending
- **type**: exception
- **confidence**: inferred
- **provenance**:
  - file: django/core/management/commands/makemigrations.py
- **rationale**: File path 'django/core/management/commands/makemigrations.py' contains exception naming pattern (legacy/compat/override)
- **target**: .wiki/domains/django/_overview.md

> makemigrations seems to contain a special case or legacy behavior

**Action** : `confirm` | `reformulate` | `reject`

### django-004

- **status**: pending
- **type**: exception
- **confidence**: inferred
- **provenance**:
  - file: django/core/management/commands/showmigrations.py
- **rationale**: File path 'django/core/management/commands/showmigrations.py' contains exception naming pattern (legacy/compat/override)
- **target**: .wiki/domains/django/_overview.md

> showmigrations seems to contain a special case or legacy behavior

**Action** : `confirm` | `reformulate` | `reject`

## base

### base-001

- **status**: pending
- **type**: business_rule
- **confidence**: inferred
- **provenance**:
  - file: tests/serializers/models/base.py
  - test: tests/serializers/models/base.py
  - comment: [TODO] Handle multiple backends with different feature flags.
- **rationale**: File 'tests/serializers/models/base.py' has tests and contains a TODO/HACK/NOTE comment
- **target**: .wiki/domains/base/_overview.md

> Handle multiple backends with different feature flags.

**Action** : `confirm` | `reformulate` | `reject`
