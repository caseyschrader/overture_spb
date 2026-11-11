# overture_spb

What is the shortest walk that hits every bar in St. Petersburg, Russia?

This project answers that with a Databricks pipeline built on [Overture Maps](https://overturemaps.org) data and the OpenStreetMap walking network. It ingests the data, cleans it in a bronze/silver/gold layout, computes walking distances between every bar, solves the route as a traveling salesman problem, and renders the result as an animated map.

**Live map:** the finished tour is in [`docs/index.html`](docs/index.html), a self-contained Leaflet page that draws the route leg by leg with a distance counter, a "next stop" panel, and play/pause/speed/scrub controls. Serve it with GitHub Pages (Settings > Pages > deploy from `/docs`) or just open the file in a browser.

## Pipeline

```
Overture Maps (S3)        OSM walk network (osmnx)
       |                           |
       v                           v
  landing volume  ------------------------
       |
       v
    bronze   raw Delta tables + ingest metadata
       |
       v
    silver   cleaned places/bars, H3 index, walk graph, bars snapped to graph nodes
       |
       v
     gold    distance matrix, tour order, route nodes, H3 cell stats
       |
       v
  animated Leaflet map
```

### 1. Ingest (`Explore Overture.ipynb`)
- Looks up the latest Overture release from the STAC catalog and reads the `divisions`, `places` and `buildings` themes straight from S3.
- Uses the city's division area to get a bounding box, then filters places and buildings to it.
- Pulls the pedestrian network with [OSMnx](https://osmnx.readthedocs.io), keeps only the largest connected component (so every node can reach every other), and collapses parallel edges to the shortest one.
- Writes everything to a Unity Catalog volume (`workspace.spb.landing`). The walk graph is also saved in this repo as `walk_nodes.parquet` and `walk_edges.parquet`.

### 2. Bronze (`to_bronze.ipynb`)
Loads the landing files into Delta tables (`bronze_places`, `bronze_buildings`, `bronze_division_area`, `bronze_walk_nodes`, `bronze_walk_edges`) and stamps each row with `ingested_at`, plus the Overture release `version` for the Overture tables.

### 3. Silver (`to_silver.ipynb`)
- Flattens the bbox structs into `lon`/`lat`.
- Keeps places with confidence >= 0.4 and dedupes by name. Bars are places with `basic_category = 'bar'`.
- Adds an H3 cell (resolution 8) to places, bars, buildings and walk nodes.
- Snaps every bar to its nearest walk node. Candidates come from the bar's H3 cell and its k=1 ring of neighbors, then the closest by haversine distance wins. `Explore Silver.ipynb` cross-checks this against a scikit-learn BallTree.
- Gives each node an integer `node_idx` and builds `silver_walk_edges` with matching indices, which is what the graph code needs.

### 4. Gold
- **`distance_matrix.ipynb`**: builds a sparse graph from the walk edges and runs batched Dijkstra from every bar node. It keeps only the bar-to-bar columns, giving a symmetric matrix of walking distances in meters (1203 distinct bar nodes), written to `gold_node_distances`. Sanity checks cover unreachable pairs, zero self-distance, symmetry, and walking vs straight-line ratio.
- **`Tour.ipynb`**: solves the tour on that matrix and builds the map.
  - Baselines: nearest neighbor, then nearest neighbor from the best starting stop.
  - Solver: Google OR-Tools routing with guided local search, warm-started from the nearest neighbor tour.
  - A minimum spanning tree weight is computed as a lower bound to judge how good the tour is.
  - The tour is expanded into the actual street-level path with Dijkstra predecessors, and every leg is checked against the matrix. The result lands in `gold_tour` and `gold_route_nodes`.
- **`Explore Silver.ipynb`**: also builds the H3 cell stats tables (`gold_cell_category`, `gold_cell_stats`, `gold_cell_top5`) with places per cell, category shares and Shannon entropy as a diversity measure.
- **`New Notebook 2026-10-23 11_29_14.ipynb`**: scratch pydeck view with an H3 density layer under the route.

## Repo layout

| File | What it is |
|---|---|
| `Explore Overture.ipynb` | Ingest from Overture and OSM into the landing volume |
| `to_bronze.ipynb` | Landing files to bronze Delta tables |
| `to_silver.ipynb` | Cleaning, H3 indexing, graph prep, bar snapping |
| `distance_matrix.ipynb` | All-pairs walking distances between bars |
| `Tour.ipynb` | TSP solve, route expansion, animated map |
| `Explore Silver.ipynb` | Checks on silver data and the gold H3 stats |
| `to_gold.ipynb` | Stub for gold tables |
| `docs/index.html` | The finished animated map |
| `walk_nodes.parquet`, `walk_edges.parquet` | Cached walking network |

## Running it

The notebooks are written for Databricks (they use `spark`, Delta tables, `displayHTML` and a Unity Catalog volume), with everything under the `workspace.spb` schema.

1. Create a cluster or serverless compute that can reach `s3://overturemaps-us-west-2` and the internet (for the OSM download).
2. Install the extras the notebooks expect: `osmnx`, `ortools`, `scikit-learn`, `pydeck`. Spark's built-in H3 functions are used for indexing.
3. Run in order: `Explore Overture` > `to_bronze` > `to_silver` > `distance_matrix` > `Tour`.

The distance matrix and the route expansion each take around a minute or two on the full graph. The OR-Tools solve is time limited (120 seconds in the notebook).

## Notes and caveats

- Overture refreshes monthly and the notebooks always pull the latest release, so bar counts and the tour will drift between runs. The release version is recorded in the bronze tables.
- Bars are deduped by name, so two different bars sharing a name collapse into one.
- Several bars can snap to the same walk node, which is why there are fewer stops than bars.
- Walking distance comes from OSM edge lengths only. There are no opening hours, no time of day, and no accounting for the Neva crossings being drawbridges at night.
- The tour is a closed loop that returns to the start.

## Data sources

- [Overture Maps Foundation](https://overturemaps.org) places, buildings and divisions (see their license terms)
- [OpenStreetMap](https://www.openstreetmap.org) contributors, via OSMnx (ODbL)
