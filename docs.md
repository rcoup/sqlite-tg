# `sqlite-tg` Documentation

As a reminder, `sqlite-tg` is still young, so breaking changes should be expected while `sqlite-tg` is in a pre-v1 stage.

<h2 name="supported-formats">Supported Formats</h2>

`tg` and `sqlite-tg` can accept geometries in [Well-known Text (WKT)](https://en.wikipedia.org/wiki/Well-known_text_representation_of_geometry), [Well-known Binary(WKB)](https://en.wikipedia.org/wiki/Well-known_text_representation_of_geometry#Well-known_binary), and [GeoJSON](https://geojson.org/) formats.

`sqlite-tg` functions will infer which format to use based on the following rules:

1. If a provided argument is a `BLOB`, then it is assumed the blob is valid WKB.
2. If the provided argument is `TEXT` and is the return value of a [JSON SQL function](https://www.sqlite.org/json1.html), or if it starts with `"{"`, then it is assumed the string is valid GeoJSON.
3. If the provided argument is still `TEXT`, then it is assumed the text is valid WKT.
4. If the provided argument is the return value of a `sqlite-tg` function that [returns a geometry pointer]()

## Pointer functions

Some functions in `sqlite-tg` use SQLite's [Pointer Passsing Interface](https://www.sqlite.org/bindptr.html) to return special objects. This is mainly done for performance benefits in specific queries, to avoid the overhead serializing/de-serializing the same geometric object multiple times.

When using one of these functions it may appear to return `NULL`. Technically it is not null, but user-facing SQL queries can't directly access the real value. Instead, other `sqlite-tg` functions can read the underlying data in their own functions. For example:

```sql
select tg_point(1, 2); -- appears to be NULL

select tg_to_wkt(tg_point(1, 2)); -- returns 'POINT(1 2)'
```

[`tg_point`](#tg-point) is a pointer function, which appears to return `NULL` when directly accessing in a query. However, it can be used to pass into other `sqlite-tg` functions, such as `tg_to_wkt()`, which can access the object under-the-hood.

All `sqlite-tg` functions that return a pointer also have counterpart functions that return "proper" values, like WKT/WKB/GeoJSON. For `tg_point()`, the counterpart functions are [`tg_point_wkt()`](#tg_point_wkt), [`tg_point_wkb()`](#tg_point_wkb), and [`tg_point_geojson()`](#tg_point_geojson).

## API Reference

All functions offered by `sqlite-tg`.

### Meta

<h4 name="tg_version"><code>tg_version()</code></h4>

Returns the version of `sqlite-tg`.

```sql
select tg_version(); -- "v0...."
```

<h4 name="tg_debug"><code>tg_debug()</code></h4>

Returns fuller debug information of `sqlite-tg`.

```sql
select tg_debug(); -- "v0....Date...Commit..."
```

### Constructors

<h4 name="tg_point"><code>tg_point(x, y)</code></h4>

A [pointer function](#pointer-functions) that returns a point geometry with the given `x` and `y` values. This value will appear to be `NULL` on direct access, and is meant for performance critical SQL queries where you want to avoid serializing/de-serializing.

In most cases, you should consider the sibling functions [`tg_point_geojson()`](#tg_point_geojson), [`tg_point_wkb()`](#tg_point_wkb), and [`tg_point_wkt()`](#tg_point_wkt).

```sql
select tg_point(1, 2); -- appears to be NULL

select tg_to_wkt(tg_point(1, 2)); -- 'POINT(1 2)'
```

<h4 name="tg_point_geojson"><code>tg_point_geojson(x, y)</code></h4>

Creates a new point geometry with the given `x` and `y` values. Returns a GeoJSON string.

```sql
select tg_point_geojson(1, 2); -- '{"type":"Point","coordinates":[1,2]}'
```

<h4 name="tg_point_wkb"><code>tg_point_wkb(x, y)</code></h4>

Creates a new point geometry with the given `x` and `y` values. Returns a WKB blob.

```sql
select tg_point_wkb(1, 2); -- X'0101000000000000000000f03f0000000000000040'
```

<h4 name="tg_point_wkt"><code>tg_point_wkt(x, y)</code></h4>

Creates a new point geometry with the given `x` and `y` values. Returns a WKT string.

```sql
select tg_point_wkt(1, 2); -- 'POINT(1 2)'
```

### Conversions

<h4 name="tg_to_geojson"><code>tg_to_geojson(geometry)</code></h4>

Converts the given geometry into a GeoJSON string. Inputs can be in [any supported formats](#supported-formats), including WKT, WKB, and GeoJSON. Based on [`tg_geom_geojson()`](https://github.com/tidwall/tg/blob/main/docs/API.md#tg_geom_geojson).

```sql
select tg_to_geojson('POINT(0 1)');
-- '{"type":"Point","coordinates":[0,1]}'

select tg_to_geojson(X'01010000000000000000000000000000000000f03f');
-- '{"type":"Point","coordinates":[0,1]}'

select tg_to_geojson('{"type":"Point","coordinates":[0,1]}');
-- '{"type":"Point","coordinates":[0,1]}'

select tg_to_geojson(tg_point(0, 1));
-- '{"type":"Point","coordinates":[0,1]}'
```

<h4 name="tg_to_wkb"><code>tg_to_wkb(geometry)</code></h4>

Converts the given geometry into a WKB blob. Inputs can be in [any supported formats](#supported-formats), including WKT, WKB, and GeoJSON. Based on [`tg_geom_wkb()`](https://github.com/tidwall/tg/blob/main/docs/API.md#tg_geom_wkb).

```sql
select tg_to_wkb('POINT(0 1)');
-- X'01010000000000000000000000000000000000f03f'

select tg_to_wkb(X'01010000000000000000000000000000000000f03f');
-- X'01010000000000000000000000000000000000f03f'

select tg_to_wkb('{"type":"Point","coordinates":[0,1]}');
-- X'01010000000000000000000000000000000000f03f'

select tg_to_wkb(tg_point(0, 1));
-- X'01010000000000000000000000000000000000f03f'
```

<h4 name="tg_to_wkt"><code>tg_to_wkt(geometry)</code></h4>

Converts the given geometry into a WKT blob. Inputs can be in [any supported formats](#supported-formats), including WKT, WKB, and GeoJSON. Based on [`tg_geom_wkt()`](https://github.com/tidwall/tg/blob/main/docs/API.md#tg_geom_wkt).

```sql
select tg_to_wkt('POINT(0 1)');
-- 'POINT(0 1)'

select tg_to_wkt(X'01010000000000000000000000000000000000f03f');
-- 'POINT(0 1)'

select tg_to_wkt('{"type":"Point","coordinates":[0,1]}');
-- 'POINT(0 1)'

select tg_to_wkt(tg_point(0, 1));
-- 'POINT(0 1)'
```

### Misc.

<h4 name="tg_type"><code>tg_type(geometry)</code></h4>

Returns a string describing the type of the provided `geometry`. Inputs can be in [any supported formats](#supported-formats), including WKT, WKB, and GeoJSON. Based on [`tg_geom_type_string()`](https://github.com/tidwall/tg/blob/main/docs/API.md#tg_geom_type_string).

Possible values:

- `"Point"`
- `"LineString"`
- `"Polygon"`
- `"MultiPoint"`
- `"MultiLineString"`
- `"MultiPolygon"`
- `"GeometryCollection"`
- `"Unknown"`

```sql
select tg_type('POINT (30 10)');
-- 'Point'
select tg_type('LINESTRING (30 10, 10 30, 40 40)');
-- 'LineString'
select tg_type('POLYGON ((30 10, 40 40, 20 40, 10 20, 30 10))');
-- 'Polygon'
select tg_type('MULTIPOINT (10 40, 40 30, 20 20, 30 10)');
-- 'MultiPoint'
select tg_type('MULTIPOLYGON (((30 20, 45 40, 10 40, 30 20)),((15 5, 40 10, 10 20, 5 10, 15 5)))');
-- 'MultiPolygon'
select tg_type('GEOMETRYCOLLECTION (POINT (40 10),LINESTRING (10 10, 20 20, 10 40),POLYGON ((40 40, 20 45, 45 30, 40 40)))');
-- 'GeometryCollection'
```

### Operations

<h4 name="tg_intersects"><code>tg_intersects(a, b)</code></h4>

Returns `1` if the `a` geometry intersects the `b` geometry, otherwise returns `0`. Will raise an error if either `a` or `b` are not valid geometries. Based on [`tg_geom_intersects()`](https://github.com/tidwall/tg/blob/main/docs/API.md#tg_geom_intersects).

The `a` and `b` geometries can be in any [supported format](#supported-formats), including WKT, WKB, and GeoJSON.

```sql
select tg_intersects(
  'LINESTRING (0 0, 2 2)',
  'LINESTRING (1 0, 1 2)'
); -- 1

select tg_intersects(
  'LINESTRING (0 0, 0 2)',
  'LINESTRING (2 0, 2 2)'
); -- 0
```

Consider this rough bounding box for San Francisco:

```
POLYGON((
  -122.51610563264538 37.81424532146113,
  -122.51610563264538 37.69618409220847,
  -122.35290547288255 37.69618409220847,
  -122.35290547288255 37.81424532146113,
  -122.51610563264538 37.81424532146113
))
```

The following SQL query, for a point within the city, returns `1`:

```sql
select tg_intersects('POLYGON((
  -122.51610563264538 37.81424532146113,
  -122.51610563264538 37.69618409220847,
  -122.35290547288255 37.69618409220847,
  -122.35290547288255 37.81424532146113,
  -122.51610563264538 37.81424532146113
))', 'POINT(-122.4075 37.787994)')
```

With a point outside the city it returns `0`:

```sql
select tg_intersects('POLYGON((
  -122.51610563264538 37.81424532146113,
  -122.51610563264538 37.69618409220847,
  -122.35290547288255 37.69618409220847,
  -122.35290547288255 37.81424532146113,
  -122.51610563264538 37.81424532146113
))', 'POINT(-73.985130 40.758896)')
```

<!--
<h4 name="tg_XXX"><code>tg_XXX()</code></h4>

```sql
select tg_XXX();
```
-->
