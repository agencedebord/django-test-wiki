---
domain: fields
confidence: llm-analyzed
last_updated: 2026-04-03
related_files:
  - docs/ref/models/fields.txt
  - django/contrib/gis/db/models/fields.py
---

# Fields

## What this domain does
The fields domain provides Django's model field system — both the standard field type library (CharField, IntegerField, etc.) and the GeoDjango spatial field hierarchy. It serves as the foundational abstraction between Python model attributes and database columns, handling type coercion, validation, serialization, and database-specific DDL generation. The spatial field subset (BaseSpatialField → GeometryField/RasterField) adds coordinate system awareness, SRID management, and geometry/raster type coercion on top of the base Field machinery. `[llm-analyzed]`

## Key behaviors
- **null vs blank: two distinct concerns**: null=True controls database storage (stores NULL), blank=True controls form validation (allows empty input). These are deliberately separate: a field can accept blank input in forms while still storing a non-null value (requiring a clean() override to supply the missing value). Django's convention for string fields is to use empty string as the 'no data' state, not NULL — so null=True on string fields is redundant and should be avoided, except when the field also has unique=True and blank=True (where NULL avoids unique constraint violations on multiple blank values). `[llm-analyzed]`
- **choices can be a callable (lazy evaluation)**: The choices argument accepts a zero-argument callable instead of a static mapping or list. This is intentional for I/O-bound choices (e.g., querying a database table for valid currencies/countries) or choices that vary per-project. The callable is invoked at form render time, not at model class definition time, enabling dynamic option sets without restarting the server. `[llm-analyzed]`
- **Spatial fields use spatial_index, not db_index**: BaseSpatialField replaces the standard db_index with a spatial_index parameter (default True). This is because spatial indexes (e.g., GIST in PostgreSQL/PostGIS) require different DDL than regular B-tree indexes. Using db_index on a geometry column would create the wrong index type. The spatial_index flag is passed to the backend's geo_db_type() to generate correct column DDL. `[llm-analyzed]`
- **SRID defaults to 4326 (WGS84) but can be -1 for unknown**: All spatial fields default to SRID 4326 (WGS84 lat/lon). SRID -1 means 'unknown/unspecified CRS'. The get_srid() method resolves SRID precedence: if the geometry object has no SRID, the field's SRID is used; if the field SRID is -1, the geometry's SRID takes priority. This allows storing geometry in any CRS while the field declares the expected system. `[llm-analyzed]`
- **SRID unit info is cached per database alias**: get_srid_info() caches unit metadata (units, units_name, spheroid WKT, geodetic flag) in a module-level _srid_cache keyed by (alias, srid). This avoids hitting the spatial_ref_sys table on every distance query construction. The cache is process-lifetime — no invalidation mechanism exists, which is fine because spatial reference systems are immutable reference data. `[llm-analyzed]`

## Domain interactions
- **Db**: Fields delegate database type generation to connection.ops.geo_db_type(self), making the actual SQL column type fully backend-dependent. The db domain's connection ops are also used for placeholder SQL (get_geom_placeholder_sql) and the Adapter wrapper for geometry serialization. `[llm-analyzed]`
- **Contrib (gis)**: BaseSpatialField and its subclasses live inside contrib.gis and depend on gdal (GDALRaster, SpatialReference) and geos (GEOSGeometry and all geometry subtypes). These are optional C-extension dependencies — if GDAL/GEOS are not installed, the gis fields are unavailable entirely. `[llm-analyzed]`
- **Core**: Spatial fields raise ImproperlyConfigured (from django.core.exceptions) when the backend does not support spatial operations, providing clear misconfiguration errors at startup rather than cryptic SQL errors at query time. `[llm-analyzed]`
- **Base, conf, tests**: These domains import from fields — base likely uses standard field types for shared abstract models, conf uses fields for settings-related models, and tests exercise field behavior. The fields domain itself has no dependencies on these consumers. `[llm-analyzed]`

## Gotchas and edge cases
- Oracle stores NULL for empty strings regardless of null=True/False — the null attribute has no effect on Oracle for empty string handling. `[llm-analyzed]`
- Using null=True on CharField/TextField creates two 'no data' states (NULL and empty string), making queries and comparisons ambiguous. Prefer null=False with blank=True for string fields. `[llm-analyzed]`
- The SRID cache (_srid_cache) is a module-level dict — it persists for the process lifetime and is never invalidated. If spatial_ref_sys data changes at runtime (rare but possible in dev), the cache will serve stale data until the process restarts. `[llm-analyzed]`
- spatial_index=True is not the same as db_index=True. Setting db_index on a geometry field will create the wrong index type. Always use spatial_index. `[llm-analyzed]`
- get_raster_prep_value() silently returns None when the is_candidate path fails GDAL conversion, but raises ValueError when a dict input fails. The silent failure path can mask data problems — callers must check for None returns. `[llm-analyzed]`

## Referenced by
- [Base](../base/_overview.md)
- [Conf](../conf/_overview.md)
- [Contrib](../contrib/_overview.md)
- [Core](../core/_overview.md)
- [Db](../db/_overview.md)
- [Django](../django/_overview.md)
- [Django-views](../django-views/_overview.md)
- [Templatetags](../templatetags/_overview.md)
- [Tests](../tests/_overview.md)
- [Utils](../utils/_overview.md)