---
domain: tests
confidence: llm-analyzed
last_updated: 2026-04-03
related_files:
  - tests/runtests.py
  - tests/test_sqlite.py
  - tests/urls.py
  - tests/requirements/py3.txt
  - tests/README.rst
---

# Tests

## What this domain does

This is Django's own test suite -- the comprehensive set of ~220 test modules that validate every aspect of the framework. The suite is driven by `tests/runtests.py`, a custom test harness that wraps Django's `DiscoverRunner` with additional features for parallel execution, test bisection, paired execution, Selenium browser testing, and per-module app installation.

Each top-level subdirectory under `tests/` is an independent test module (effectively a mini Django app) that typically contains its own `models.py`, `tests.py`, and sometimes `urls.py`, `views.py`, fixtures, and templates. Test modules are auto-discovered by scanning subdirectories that contain an `__init__.py`. `[llm-analyzed]`

## Key behaviors

### Test runner (`runtests.py`)

- **Auto-discovery**: scans `tests/` for subdirectories containing `__init__.py`, skipping `import_error_package` and `test_runner_apps` by design.
- **Dynamic app installation**: each test module is temporarily added to `INSTALLED_APPS` before tests run, along with its contrib dependencies (mapped via `CONTRIB_TESTS_TO_APPS`).
- **Migrations disabled**: auth, contenttypes, and sessions migrations are set to `None` -- tables are created directly via `syncdb`-style creation for speed.
- **Parallel execution**: supports `--parallel` (auto-detects core count), cloning databases when the backend supports it. Uses `tblib` for serializing tracebacks across processes.
- **Bisection mode** (`--bisect`): binary-searches test modules to isolate cross-module pollution causing a failure.
- **Paired mode** (`--pair`): runs every module paired with a suspect module to find interaction bugs.
- **Shuffle** (`--shuffle`): randomizes test ordering to catch order-dependent tests.
- **Selenium support** (`--selenium`): enables browser-driven tests tagged `selenium`; supports headless mode, remote hubs, and screenshots.
- **Deprecation enforcement**: `RemovedInDjango70Warning` is turned into an error, preventing deprecated API usage from going unnoticed.
- **GC tuning**: threshold raised to 100,000 (from default 700) on CPython to reduce collection overhead during the suite run.

### Test base classes used across the suite

- `TestCase` -- wraps each test in a transaction that is rolled back (fast, isolated). Requires `available_apps = None`.
- `TransactionTestCase` -- actually commits transactions; **must** declare `available_apps` or `runtests.py` raises an error to force test isolation.
- `SimpleTestCase` -- no database access, used for pure-logic or HTTP-level tests.
- `SeleniumTestCase` -- browser-based functional tests, tagged `selenium`, excluded from default runs.

### Test module structure conventions

- Each module is a self-contained Django app with its own models, typically defining only the models needed for its specific area (e.g., `basic/models.py` defines `Article`, `SelfRef`).
- Test files import from `.models` using relative imports.
- Many modules have a `_regress` counterpart (e.g., `aggregation_regress`, `defer_regress`) that covers edge cases and regression bugs separately from the main module.

### Default settings (`test_sqlite.py`)

- Uses SQLite in-memory for both `default` and `other` database aliases.
- `MD5PasswordHasher` for speed.
- `USE_TZ = False` by default (timezone tests override this).

## Domain interactions

- **django.test framework** (`django/test/`): the suite is the primary consumer of `TestCase`, `TransactionTestCase`, `DiscoverRunner`, `override_settings`, `CaptureQueriesContext`, and all test utilities.
- **django.contrib apps**: test modules for contrib features (auth, admin, sessions, staticfiles, GIS, etc.) live here and exercise those contrib packages end-to-end.
- **Database backends** (`tests/backends/`): backend-specific tests validate connection handling, schema operations, and feature flags across SQLite, PostgreSQL, MySQL, and Oracle.
- **GIS tests** (`tests/gis_tests/`): nested test app structure, only discovered when the backend reports `gis_enabled = True`. Requires `django.contrib.gis`.
- **Migrations** (`tests/migrations/`, `tests/migrations2/`): test the migration framework itself, including custom migration operations.

## Gotchas and edge cases

- **`available_apps` enforcement**: `runtests.py` monkey-patches `TransactionTestCase.available_apps` to raise if not declared. Forgetting this in a new `TransactionTestCase` subclass causes a cryptic `Exception` rather than a test failure.
- **Module isolation**: each test module is installed as a Django app. Two modules defining a model with the same `db_table` or `app_label` can clash. `model_inheritance_same_model_name` is explicitly excluded from bisect/pair runs for this reason.
- **GIS gating**: requesting `gis_tests` on a non-GIS backend causes an immediate `sys.exit(1)`, not a skip.
- **Selenium tests are excluded by default**: they only run when `--selenium` is passed, which auto-adds the `selenium` tag filter.
- **`--start-at` / `--start-after` are mutually exclusive with test labels**: passing both causes an abort.
- **Temporary directory**: the suite creates a dedicated `TMPDIR` per run and cleans it up via `atexit`. Tests relying on temp files inherit this directory.
- **Import error package**: `tests/import_error_package/` is intentionally broken (for testing import error handling) and is always skipped during discovery.

## Dependencies

- **Python standard library**: `unittest`, `argparse`, `multiprocessing`, `tempfile`, `gc`
- **Django framework**: the entire framework is the system under test
- **Third-party (optional, via `requirements/py3.txt`)**: `selenium`, `Pillow`, `PyYAML`, `numpy`, `redis`, `pylibmc`, `pymemcache`, `argon2-cffi`, `bcrypt`, `geoip2`, `jinja2`, `tblib`, `sqlparse`, `pywatchman`
- **Database backends**: SQLite (default), PostgreSQL (`requirements/postgres.txt`), MySQL (`requirements/mysql.txt`), Oracle (`requirements/oracle.txt`)

## Referenced by

- CI pipelines and contributor workflows (the suite is the primary quality gate for Django development)
- `django/test/` -- the test infrastructure that this suite exercises
- `.github/` -- CI configuration that invokes `runtests.py`
