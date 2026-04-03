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
The fields domain handles Django's model field system, with a particular focus here on GIS/spatial fields (GeoDjango). It bridges the gap between Python geometric objects and spatial database columns, managing coordinate system metadata, geometry type enforcement, and database-specific SQL generation. The core pattern is that spatial fields extend Django's standard Field infrastructure but override the database preparation pipeline to route values through spatial adapters that the database backend (PostGIS, SpatiaLite, etc.) understands. `[llm-analyzed]`

## Key behaviors
- **SRID metadata is cached per database alias to avoid repeated queries**: get_srid_info() maintains a module-level _srid_cache keyed by (connection.alias, srid). The first call per SRID per connection hits spatial_ref_sys (or falls back to GDAL's SpatialReference if the backend doesn't expose that table). Subsequent calls return the cached SRIDCacheEntry namedtuple containing units, units_name, spheroid WKT, and a geodetic flag. This matters for distance query construction, which needs unit information frequently. `[llm-analyzed]`
- **SRID defaults to 4326 (WGS84) and is always included in migration output**: BaseSpatialField.__init__ defaults srid=4326. In deconstruct(), srid is always written to kwargs unconditionally — unlike spatial_index which is omitted when it equals the True default. The comment says this is for 'less fragility', meaning it avoids silent SRID changes if the default ever changes. `[llm-analyzed]`
- **spatial_index replaces db_index for GIS fields**: Spatial fields use spatial_index=True (default) instead of Django's standard db_index. This is because spatial indexes (e.g., GIST in PostGIS) are created via database-specific DDL that differs from regular B-tree indexes. The spatial_index flag is passed to the backend's geo_db_type() indirectly and handled by the backend's spatial index creation logic. `[llm-analyzed]`
- **Geometry value prep handles multiple input formats with fallback logic**: get_prep_value() accepts GEOSGeometry objects directly, or falls back to attempting raster/geometry construction from bytes, strings, or dicts. The is_candidate check distinguishes between 'maybe a geometry string' (bytes/str) and 'definitely trying to be a raster' (dict). A dict that fails GDALRaster construction raises ValueError explicitly, while a bytes/str failure silently returns None from get_raster_prep_value — the caller must handle the None case. `[llm-analyzed]`
- **Geography flag conditionally wraps values in geography-aware adapter**: get_db_prep_value() checks both self.geography and connection.features.supports_geography before passing {'geography': True} to the connection's Adapter. This means the same field definition behaves differently across backends — on backends without geography support, geometry is used instead. The geography flag itself is defined on GeometryField (subclass), not BaseSpatialField. `[llm-analyzed]`

## Domain interactions
- **Django (core DB backend)**: BaseSpatialField.db_type() delegates entirely to connection.ops.geo_db_type(self), meaning the SQL column type is determined by the backend, not the field. Similarly, get_placeholder_sql() delegates to connection.ops.get_geom_placeholder_sql(). The field itself is backend-agnostic; all database-specific decisions are pushed to the connection's ops object. `[llm-analyzed]`
- **Proxy**: The GIS fields use SpatialProxy (from the proxy domain) as a descriptor to intercept attribute access on model instances. This allows lazy conversion of raw database values (WKB bytes, strings) into GEOSGeometry or GDALRaster objects on first access, rather than at load time. `[llm-analyzed]`
- **Gdal / geos (django.contrib.gis)**: get_prep_value() and get_raster_prep_value() directly construct GDALRaster and GEOSGeometry objects. The field catches GDALException during raster construction as part of its type-detection logic. GEOSGeometry is the canonical Python representation for vector geometry; GDALRaster for raster data. `[llm-analyzed]`
- **Expressions**: Spatial fields participate in the ORM expression system through get_db_prep_value(), which wraps values in backend-specific Adapter objects before they reach the SQL compiler. This is how spatial values get encoded as WKB or geography types in parameterized queries. `[llm-analyzed]`
- **Base (model base)**: Fields are registered on models via contribute_to_class() (inherited from Field). The GIS field subclasses rely on the standard field registration pipeline but add spatial index creation as a side effect handled by the backend's schema editor, not the field itself. `[llm-analyzed]`

## Gotchas and edge cases
- The _srid_cache is a module-level global dict. In long-running processes with multiple databases, stale entries won't be invalidated if the spatial_ref_sys table changes — a server restart is required to pick up SRID changes. `[llm-analyzed]`
- get_raster_prep_value() silently returns None for bytes/str inputs that fail GDALRaster construction (is_candidate=True path). The caller (get_prep_value) must handle None and continue attempting geometry construction. If both raster and geometry construction fail, the raw value is passed through — which will likely cause a database error rather than a clean Python exception. `[llm-analyzed]`
- BaseSpatialField sets empty_strings_allowed = False. This means the ORM will not substitute '' for None in string coercion paths. Spatial fields must always use NULL for missing values, not empty strings — the null=False + blank=True pattern that works for text fields is not viable here. `[llm-analyzed]`
- spatial_index is decoupled from db_index. Setting db_index=True on a spatial field does not create a spatial index — it creates a regular B-tree index on the geometry column, which is almost certainly wrong. Only spatial_index=True triggers the correct GIST/R-tree index creation. `[llm-analyzed]`
- The geodetic() method returning True means distance calculations must use degrees or a spheroid-aware function. Passing meter-based distance values to a query on a geodetic field will silently compute incorrect results — the ORM does not automatically convert units. `[llm-analyzed]`

## Dependencies
- [Base](../base/_overview.md)
- [Constraints](../constraints/_overview.md)
- [Django](../django/_overview.md)
- [Expressions](../expressions/_overview.md)
- [Indexes](../indexes/_overview.md)
- [Lookups](../lookups/_overview.md)
- [Options](../options/_overview.md)
- [Proxy](../proxy/_overview.md)

## Referenced by
- [Base](../base/_overview.md)
- [Django](../django/_overview.md)
- [Tests](../tests/_overview.md)