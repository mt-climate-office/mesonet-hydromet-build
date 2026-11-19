# Vendored geometry for the HydroMet build map

These files ship with the page so the grid draws immediately and the map does
not go blank when the CDN is slow. The optional county / watershed / tribal-land
overlays are *not* vendored — they stream from
`https://data.climate.umt.edu/mesonet/fgb/` on demand.

## `mt_grids_simple.geojson`

The Montana ACE grid cells, cut from the basin-wide grid
`data/umrb-ace-grid.geojson` in the sibling
[umrb-ace](https://github.com/mt-climate-office/umrb-ace) repo (540 cells across
MT, WY, ND, SD and NE, with `Cell` and `State` properties). Only `State = 'MT'`
is kept, and only the `Cell` property. Run from a directory beside both repos:

```sh
ogr2ogr -f GeoJSON -lco COORDINATE_PRECISION=5 -lco RFC7946=YES \
  -where "State = 'MT'" -select Cell \
  mt_grids_raw.geojson ../umrb-ace/data/umrb-ace-grid.geojson
# -target 1 selects the polygon layer; RFC7946 conversion emits two stray
# polyline records that mapshaper would otherwise write to a second file.
npx mapshaper@0.7.101 -i mt_grids_raw.geojson -target 1 \
  -simplify visvalingam 5% keep-shapes \
  -o precision=0.00001 format=geojson mt_grids_simple.geojson
```

205 cells, ~358 KB (~80 KB gzipped). Most cells are simple quads; the size comes
from the handful clipped to the basin boundary. 5% simplification is well below
what is visible at this map's zoom range — drop it further only if the clipped
edges start to matter. mapshaper reports 7 self-intersections it cannot repair;
two cells end up technically invalid, which MapLibre draws without complaint.

Regenerate whenever the umrb-ace grid changes. The page keys cells by the
`Cell` property (`H-8` style, unpadded), which is also the registry's
`ace_grid` format.

Earlier versions were cut from `https://data.climate.umt.edu/mesonet/fgb/mt_grids.fgb`
(last modified 2023-09-22). That file is out of date: it has a `J-13` the
current grid dropped, and lacks the `E-9` that `acesfork` sits in.

## `mt_grids_full_simple.geojson`

The statewide grid, drawn only in the internal view (`?internal=1`). It uses the
same lattice as the UMRB grid: squares of √500 mi on Alex Stum's Albers
parameterization (`umrb-grid.R` in umrb-ace). They are clipped to the Montana
line, and a square is kept only if at least 40% of it lies in Montana.
The geometry is umrb-ace's `data/mt-ace-grid-clipped.geojson`. Its `cell`
column predates the 2026-10-21 grid revision, so it is ignored, and every
cell's ID is recomputed from its lattice position:

- **Columns** are letters, with A at the UMRB's west edge (A…Z, AA…AL). The
  six columns west of A are `W-A` (next to A) out to `W-F` (the Idaho line).
- **Rows** are numbers, with 1 at the top of the UMRB. The one row above it is
  `0`. Only `W-D-0` and `W-E-0` fall in it.
- IDs are upper case. Lower case is reserved, as in umrb-ace, for a cell
  dropped from consideration. None is dropped yet.

The result is 294 cells: the 205 UMRB cells, the 55 west of A, and 34
in-lattice cells outside the UMRB build (A-3, J-13, …). The 205 UMRB IDs are
the same as in `mt_grids_simple.geojson`, but here they are full squares rather
than basin-clipped ones. Run from a directory beside both repos:

```sh
ALBERS='+proj=aea +lat_0=41.8865 +lat_1=43.0 +lat_2=47.8 +lon_0=-104.0487 +units=m +datum=NAD83'
ogr2ogr -f GPKG mt_grid_albers.gpkg -t_srs "$ALBERS" -nln g \
  ../umrb-ace/data/mt-ace-grid-clipped.geojson
# 35986.03 m = √500 mi. k = 0 is column A; n = 1 is row 1.
ogr2ogr -f GeoJSON -lco COORDINATE_PRECISION=5 -lco RFC7946=YES -t_srs EPSG:4326 \
  -dialect sqlite -sql "
  SELECT CASE WHEN k >= 26 THEN 'A' || char(39 + k)
              WHEN k >= 0  THEN char(65 + k)
              ELSE 'W-' || char(64 - k) END || '-' || n AS Cell, geom
  FROM (SELECT CAST(floor(ST_X(ST_PointOnSurface(geom)) / 35986.03) AS INTEGER) + 19 AS k,
               23 - CAST(floor(ST_Y(ST_PointOnSurface(geom)) / 35986.03) AS INTEGER) AS n,
               geom FROM g)" \
  mt_grids_full_raw.geojson mt_grid_albers.gpkg
npx mapshaper@0.7.101 -i mt_grids_full_raw.geojson -target 1 \
  -simplify visvalingam 5% keep-shapes \
  -o precision=0.00001 format=geojson mt_grids_full_simple.geojson
```

After regenerating, check that every ID in `mt_grids_simple.geojson` appears
here, and that each of those cells lies inside the same-named square. The page
upper-cases every ID it reads, so a lower-case `Cell` would still draw. Drop a
cell by deleting its feature here, just as umrb-ace keeps dropped cells out of
its geojson. Regenerate when umrb-ace's statewide grid changes.

## `mt_state_simple.geojson`

Montana's boundary, byte-identical to `data/mt_state_simple.geojson` in the
sibling [mesonet-maintenance](https://github.com/mt-climate-office/mesonet-maintenance) repo. Keep the two
in sync if either is ever re-exported.
