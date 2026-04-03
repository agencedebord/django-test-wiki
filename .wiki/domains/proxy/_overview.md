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
SpatialProxy is a lazy-loading descriptor for geographic field values (geometries and rasters) on Django model instances. It extends Django's DeferredAttribute to intercept attribute access and transparently convert raw stored values (WKT strings, HEX, WKB bytes, JSON/dict) into proper geometry or raster objects on first access, caching the result back on the instance. It also enforces type safety on assignment. `[llm-analyzed]`

## Key behaviors
- **Lazy instantiation with caching**: On first __get__, if the stored value is a raw string/bytes (not already an instance of the target class), SpatialProxy calls _load_func(geo_value) to construct the geometry/raster object, then writes the result back into instance.__dict__[field.attname]. Subsequent accesses hit the dict directly, bypassing the descriptor entirely — a standard Python descriptor short-circuit pattern. `[llm-analyzed]`
- **Deferred DB loading fallback**: If the field's attname is not in instance.__dict__ (e.g., it was deferred by a QuerySet.defer() call), SpatialProxy falls back to super().__get__() which triggers Django's DeferredAttribute machinery to fetch the value from the database. This means spatial fields participate correctly in deferred loading. `[llm-analyzed]`
- **None and empty string both map to None**: Both None and the empty string '' are treated as absent geometry and return None from __get__. This accommodates databases or serializers that may represent missing geometries as empty strings rather than SQL NULL. `[llm-analyzed]`
- **SRID auto-assignment on geometry set**: In __set__, when assigning a geometry instance whose srid is None, SpatialProxy automatically sets value.srid = self.field.srid. This ensures geometries without an explicit SRID inherit the field's declared SRID rather than failing silently or causing downstream CRS mismatches. `[llm-analyzed]`
- **Raster and geometry assignment paths are divergent**: The __set__ method branches on field.geom_type == 'RASTER' first. Rasters accept None, str, dict, or raster instances. Geometries accept None, str (WKT/HEX), memoryview (WKB), or geometry instances. Any other type raises TypeError immediately. This means the same descriptor class handles two fundamentally different geographic data types with separate validation rules. `[llm-analyzed]`

## Domain interactions
- **Django**: Inherits from DeferredAttribute (django.db.models.query_utils). Relies on its __get__ implementation to handle database fetching when the value has been deferred. The attname and srid attributes are read from the field object passed at construction, meaning SpatialProxy is tightly coupled to Django's field interface. `[llm-analyzed]`
- **Fields**: Consumed by geographic field classes (GeometryField, RasterField, etc.) which instantiate SpatialProxy with themselves as the field argument and pass their geometry/raster class as klass. The proxy is installed as the field's descriptor on the model class via contribute_to_class. `[llm-analyzed]`

## Gotchas and edge cases
- Assigning a raw string or WKB to the field stores it as-is in instance.__dict__. The conversion to a geometry object only happens on the next __get__ call. Code that writes then immediately reads instance.__dict__[field.attname] directly (bypassing the descriptor) will see the unconverted raw value. `[llm-analyzed]`
- The SRID auto-assignment in __set__ mutates the value object in place (value.srid = self.field.srid). If the same geometry object is shared across multiple model instances or fields with different SRIDs, the last assignment wins and all references see the mutated SRID. `[llm-analyzed]`
- Empty string '' is silently normalized to None on read. If a field stores '' in the database, __get__ returns None — there is no distinction between NULL and empty string at the Python layer. `[llm-analyzed]`
- The TypeError raised in __set__ for unsupported types includes the class name and geom_type but not the field name, which can make debugging harder when a model has multiple spatial fields. `[llm-analyzed]`
- load_func defaults to klass itself (i.e., klass(geo_value)). If the class constructor has a different signature than a simple single-argument call, a custom load_func is required — there is no fallback or error hint if the constructor fails. `[llm-analyzed]`

## Dependencies
- [Base](../base/_overview.md)
- [Constraints](../constraints/_overview.md)
- [Django](../django/_overview.md)
- [Expressions](../expressions/_overview.md)
- [Indexes](../indexes/_overview.md)
- [Lookups](../lookups/_overview.md)
- [Options](../options/_overview.md)

## Referenced by
- [Base](../base/_overview.md)
- [Django](../django/_overview.md)
- [Fields](../fields/_overview.md)
- [Tests](../tests/_overview.md)