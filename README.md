# Spatial Analysis Concepts: from Alteryx to Python

**Brown Belt I · Class 19 · Spatial Tools**

A beginner-friendly Jupyter notebook, [`spatial_analysis_concepts.ipynb`](spatial_analysis_concepts.ipynb), that re-creates the Alteryx Designer spatial tools from class in Python. It uses a tiny made-up dataset, **3 coffee shops and 8 customers in Chicago**, so every result fits on one screen.

Each section pairs:

1. a plain-language explanation that names the **matching Alteryx tool**, and
2. a short Python cell you can run, change and re-run.

No Python expertise needed: run the cells top to bottom with **Shift + Enter** and read the comments.

---

## What's in this folder

```text
brown-belt-1/
├── spatial_analysis_concepts.ipynb         # the notebook (outputs already saved)
├── README.md
├── LICENSE
└── .gitignore                              # keeps secret/ and .venv/ out of git
```

The class slides and the Alteryx reading (*What Is Spatial Analysis?*) are kept in a local `secret/` folder and are **not** in this repo; they're shared in class.

---

## The big idea: location is data

Spatial analysis treats a location as something you can **calculate with** and **join to business data**, not just a dot on a map. The notebook follows the five-step workflow from the reading:

```mermaid
flowchart LR
    A["1 · Prepare<br/>clean lat/long"] --> B["2 · Represent<br/>points & polygons"]
    B --> C["3 · Relate<br/>distance, nearest,<br/>inside, trade area"]
    C --> D["4 · Calculate<br/>new columns"]
    D --> E["5 · Combine<br/>with business data"]
```

---

## Notebook map

```mermaid
flowchart TD
    S1["1 · What is spatial analysis?"] --> S2["2 · Create Points<br/>+ polygons"]
    S2 --> S3["3 · Coordinate systems<br/>degrees vs metres"]
    S3 --> S4["4 · Distance"]
    S3 --> S5["5 · Find Nearest"]
    S3 --> S6["6 · Trade Area"]
    S3 --> S7["7 · Spatial Match"]
    S6 --> S7
    S4 --> S8["8 · Map output"]
    S5 --> S8
    S7 --> S8
    S5 --> S9["9 · Spatial + business data"]
    S9 --> T["Summary table<br/>Alteryx → Python"]
```

| # | Section | Alteryx tool | Question it answers |
|---|---|---|---|
| 1 | What is spatial analysis? | — | Why location matters for business questions |
| 2 | Create Points | **Create Points** | How do two number columns become a point on the map? |
| 3 | Coordinate systems | *(coordinate settings)* | Why can't I measure distance in degrees? |
| 4 | Distance | **Distance** | How far is each customer from Store A, or from a territory border? |
| 5 | Find Nearest | **Find Nearest** | Which store is closest to each customer, within 5 km? |
| 6 | Trade Area | **Trade Area** | What area does each store serve, with and without overlap? |
| 7 | Spatial Match | **Spatial Match** | Which territory is each customer in? Which customers are in each trade area? |
| 8 | Map output | **Browse** | What does it all look like on an interactive map? |
| 9 | Putting it together | — | Which store “owns” the most customer spend? |

---

## How the data flows

Two small tables become spatial tables, get converted to metres, then feed every tool:

```mermaid
flowchart LR
    ST["stores table<br/>decimal lat/long"] --> CP1["Create Points"]
    CU["customers table<br/>lat/long × 1,000,000<br/>+ spend"] --> DIV["÷ 1,000,000"] --> CP2["Create Points"]
    TE["2 territory<br/>rectangles"] --> PG["Polygons"]

    CP1 --> G[("GeoDataFrames<br/>EPSG:4326, degrees")]
    CP2 --> G
    PG --> G
    G -- ".to_crs('EPSG:26916')" --> M[("GeoDataFrames<br/>UTM 16N, metres")]

    M --> D["Distance"]
    M --> N["Find Nearest"]
    M --> TA["Trade Area"]
    M --> SM["Spatial Match"]
    TA --> SM
    N --> BUS["Spend per store"]
    G --> MAP["folium map"]
    TA --> MAP
```

> **Why convert to metres?** Latitude and longitude are angles. A degree of longitude gets shorter as you move away from the equator, so distances and buffers need a **projected** coordinate system in metres. The notebook uses `EPSG:26916` (UTM zone 16N), which is accurate around Chicago.

---

## The tools at a glance

### Distance and Find Nearest

```mermaid
flowchart LR
    C(("Customer")) -- "distance to point" --> S(("Store"))
    C -- "distance to polygon<br/>0 if inside" --> P["Territory"]
    C -- "distance to edge<br/>(boundary)" --> P
    C -- "Find Nearest<br/>top N within max distance" --> U["All stores"]
```

Customer 8 is more than 5 km from every store, so Find Nearest returns **no match**. Section 9 flags that gap as a possible site for a new store.

### Trade Area: eliminating overlap

```mermaid
flowchart LR
    P(("Store point")) -- ".buffer(3000)" --> R["3 km circle"]
    R --> O{"Eliminate<br/>overlap?"}
    O -- No --> A["Circles may overlap:<br/>a customer can sit in two"]
    O -- Yes --> V["Cut each circle with its<br/>Voronoi cell (closer to this<br/>store than any other)"]
    V --> B["Each customer belongs<br/>to exactly one store"]
```

### Spatial Match: choosing an action

```mermaid
flowchart TD
    Q{"How should the two<br/>shapes relate?"}
    Q -- "share some area" --> I["Intersects"]
    Q -- "target fully holds universe" --> C["Contains"]
    Q -- "target fully inside universe" --> W["Within"]
    Q -- "only boundaries meet" --> T["Touches"]
    Q -- "share area OR boundaries meet" --> TI["Touches or Intersects"]
    Q -- "fast, rough check" --> BR["Bounding Rectangle<br/>Overlaps"]
```

| Alteryx action | Python (shapely) | Notebook example |
|---|---|---|
| Intersects | `a.intersects(b) and not a.touches(b)` | overlapping squares |
| Contains | `a.contains(b)` | customers inside each trade area |
| Within | `a.within(b)` | territory of each customer |
| Touches | `a.touches(b)` | squares side by side |
| Touches or Intersects | `a.intersects(b)` | |
| Bounding Rectangle Overlaps | `a.envelope.intersects(b.envelope)` | L-shape vs. far square: boxes overlap, shapes don't |

---

## Alteryx → Python cheat sheet

```mermaid
flowchart LR
    subgraph Alteryx
        A1[Create Points]
        A2[Distance]
        A3[Find Nearest]
        A4[Trade Area]
        A5[Spatial Match]
        A6[Browse]
    end
    subgraph Python
        P1["gpd.points_from_xy"]
        P2[".distance()"]
        P3["gpd.sjoin_nearest"]
        P4[".buffer() + Voronoi"]
        P5["gpd.sjoin(predicate=…)"]
        P6[".plot() / folium.Map"]
    end
    A1 --> P1
    A2 --> P2
    A3 --> P3
    A4 --> P4
    A5 --> P5
    A6 --> P6
```

| Python library | Plays the role of… |
|---|---|
| `pandas` | an Alteryx data stream (a table) |
| `shapely` | single shapes: points, polygons, distances |
| `geopandas` | a **spatial** data stream (a table with a geometry column) |
| `pyproj` | coordinate system conversions (used inside geopandas) |
| `matplotlib`, `folium` | the Browse tool's map view (static and interactive) |

---

## Run it

The notebook already contains its outputs, so you can **read it on GitHub or in VS Code without running anything**. To run it yourself:

First get the code:

```bash
git clone https://github.com/iportilla/brown-belt-1.git
cd brown-belt-1
```

### Option A: you already have a `.venv` with the libraries

(For example, the one in the instructor's original folder. A fresh clone doesn't include one; use Option B.) The `.venv` has every library the notebook needs, but not the JupyterLab app.

- **VS Code:** open the notebook and pick `.venv` as the kernel (top-right **Select Kernel**). Nothing else to install.
- **JupyterLab in the browser:** install it once, then start it:

```bash
source .venv/bin/activate
pip install jupyterlab        # first time only
jupyter lab spatial_analysis_concepts.ipynb
```

### Option B: a fresh setup

**macOS / Linux**

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install geopandas shapely pyproj matplotlib folium jupyterlab
jupyter lab spatial_analysis_concepts.ipynb
```

**Windows (PowerShell)**

```powershell
py -m venv .venv
.venv\Scripts\Activate.ps1
pip install geopandas shapely pyproj matplotlib folium jupyterlab
jupyter lab spatial_analysis_concepts.ipynb
```

If PowerShell says scripts are disabled, run `Set-ExecutionPolicy -Scope CurrentUser RemoteSigned` once and activate again.

Tested with Python 3.12, geopandas 1.2, shapely 2.2, pandas 3.1 and folium 0.20.

---

## Try this next

- **Change the radius**: set `RADIUS_M` in section 6 to 1,500 or 5,000 and watch which customers each store keeps.
- **Change the search limit**: raise `max_distance` in section 5 so customer 8 gets a nearest store.
- **Add a store**: put a fourth store near customer 8 and re-run everything. How does spend per store change?
- **Break the coordinates**: swap latitude and longitude for one store and see where it lands on the map. This is the most common beginner mistake.

**Not covered** (but mentioned in the reading): geocoding addresses into coordinates, drive-time distance along real roads, and clustering / hotspot analysis.

---

## Troubleshooting

| Problem | Fix |
|---|---|
| `ModuleNotFoundError: geopandas` | The notebook isn't using `.venv`. Pick it as the kernel, or activate it before starting Jupyter. |
| Points show up in the ocean or Antarctica | Latitude and longitude are swapped. A point is `(longitude, latitude)`. |
| Distances look like 0.01 | You measured in degrees. Convert with `.to_crs("EPSG:26916")` first. |
| The interactive map is blank | Folium loads map tiles from the internet; check your connection or trust the notebook in Jupyter. |
| Layers don't line up | They're in different coordinate systems. Compare `gdf.crs` and convert one to match. |
