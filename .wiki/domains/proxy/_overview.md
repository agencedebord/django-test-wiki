---
domain: proxy
confidence: llm-analyzed
last_updated: 2026-04-03
related_files:
  - tests/auth_tests/models/proxy.py
  - django/contrib/gis/db/models/proxy.py
---

# Proxy

## What this domain does
SpatialProxy is a Python descriptor that provides lazy instantiation and type-coercion for geographic model fields (geometry and raster). It extends Django's DeferredAttribute to intercept get/set access on model instances, converting raw database values (WKT, HEX, WKB, JSON) into proper geometry/raster objects on first access, and caching the result back on the instance dict to avoid repeated conversions. `[llm-analyzed]`

## Key behaviors
- **Lazy geometry/raster instantiation on first access**: When a geographic field is accessed on a model instance, SpatialProxy checks the instance's __dict__ for a cached value. If the stored value is already the target class, it's returned directly. If it's a raw value (WKT, HEX, EWKB string, etc.), it calls self._load_func(geo_value) to instantiate the geometry/raster object, then writes the instantiated object back to instance.__dict__[field.attname], so subsequent accesses skip conversion entirely. `[llm-analyzed]`
- **Separate load function from class**: The constructor accepts an optional load_func parameter distinct from klass. This allows the actual parsing/loading logic to differ from the class used for isinstance checks. If load_func is not provided, klass itself is used as the constructor. This separation supports cases where the factory function is not the class constructor (e.g., a registry lookup or format-aware parser). `[llm-analyzed]`
- **SRID auto-assignment on geometry set**: When setting a geometry value that is already an instance of the target class but has no SRID (srid is None), SpatialProxy automatically assigns the field's configured SRID to the geometry object. This prevents SRID mismatches when geometries are constructed without explicit spatial reference information. `[llm-analyzed]`
- **Raster vs geometry assignment validation**: The __set__ method branches on the field's geom_type == 'RASTER'. For raster fields, accepted types are None, str, dict, or the raster class. For geometry fields, accepted types are None, str, memoryview (WKB binary), or the geometry class. Any other type raises a TypeError with a descriptive message including the instance class name, geometry type, and offending value type. `[llm-analyzed]`
- **Empty string treated as null geometry**: In __get__, an empty string ('') is explicitly treated the same as None and returns geo_obj = None. This handles cases where the database or serialization layer produces an empty string instead of NULL for absent geometries. `[llm-analyzed]`

## Domain interactions
- **Django/db/models/query_utils.DeferredAttribute**: SpatialProxy extends DeferredAttribute, inheriting its deferred-loading mechanism. The super().__get__() fallback is called when the value is not present in instance.__dict__ (i.e., field was deferred in a queryset), triggering a database fetch before the geo conversion is applied. `[llm-analyzed]`
- **Contrib/gis geometry and raster classes (GEOS, OGR, GDALRaster)**: The _klass parameter is the target geometry or raster class passed in at field construction time. SpatialProxy is generic over this class — it does not import geometry types directly, keeping it decoupled from specific GIS backends. `[llm-analyzed]`
- **Geographic model fields (GeometryField, RasterField)**: The proxy reads self.field.attname, self.field.geom_type, and self.field.srid from the field object it is bound to. It is instantiated by geographic field descriptors, not used standalone. `[llm-analyzed]`

## Gotchas and edge cases
- The write-back caching in __get__ (setattr on the instance dict) means the first access is destructive: the raw string/bytes value stored in the dict is replaced by the instantiated object. Code that reads instance.__dict__[field.attname] directly after first access will see the object, not the original raw value. `[llm-analyzed]`
- SRID auto-assignment only applies when the incoming value is already an instance of _klass with srid=None. Raw string inputs (WKT, HEX) are passed directly to load_func without SRID injection — the geometry parser must handle SRID from the string content itself (e.g., EWKT prefix 'SRID=4326;...'). `[llm-analyzed]`
- The raster branch in __set__ accepts dict values (for JSON-encoded rasters), but the geometry branch does not — passing a dict to a geometry field will raise TypeError. The asymmetry is intentional and matches the different serialization formats of the two types. `[llm-analyzed]`
- memoryview is accepted in __set__ for geometry fields (representing WKB binary), but not for raster fields. Passing raw binary to a raster field raises TypeError. `[llm-analyzed]`
- Because SpatialProxy inherits DeferredAttribute, it participates in Django's deferred field system. A geometry field that was deferred (via .defer() or .only()) will trigger a new DB query on first access, then cache the instantiated geometry object — the deferred-load and the geo-conversion are both transparent to the caller. `[llm-analyzed]`

## Dependencies
- [Base](../base/_overview.md)
- [Constraints](../constraints/_overview.md)
- [Contrib](../contrib/_overview.md)
- [Db](../db/_overview.md)
- [Django](../django/_overview.md)
- [Expressions](../expressions/_overview.md)
- [Indexes](../indexes/_overview.md)
- [Lookups](../lookups/_overview.md)
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
- [Templatetags](../templatetags/_overview.md)
- [Tests](../tests/_overview.md)
- [Utils](../utils/_overview.md)