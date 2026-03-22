# AstroPlannerData — DSO Catalog Generation Notebook

## Goal
Build a clean Jupyter notebook that generates `dso.json` — the DSO catalog consumed by AstroPlanner via `Config.DSO_URL`.

## Output format (must match app's `DsoResponse`)
```json
[
  {
    "displayName": "Orion Nebula",
    "objectId": "m42",
    "ra": 83.8221,
    "dec": -5.3911,
    "type": "nebula",
    "subType": "Emission Nebula",
    "constellation": "Orion",
    "recommended": true,
    "angularSizeMajor": 65.0,
    "angularSizeMinor": 65.0,
    "magnitude": 4.0
  }
]
```

**Valid `type` values** (must match `ObjectType` enum in AstroPlanner):

| `type` | App enum |
|---|---|
| `"galaxy"` | `GALAXY` |
| `"nebula"` | `NEBULA` |
| `"cluster"` | `CLUSTER` |
| `"star"` | `STAR` |

`type` is always derived from `subType` — never set manually.

---

## One-time bootstrap
Load the existing `dso.json` to populate the Cell 1 array. Extract `objectId`, `displayName`, and `recommended` from each record. After this step, Cell 1 becomes the static source of truth and `dso.json` is output-only.

---

## Notebook structure

### Cell 1 — Object list (primary input)
The only cell that needs editing to add a new object:
```python
objects = [
    {"objectId": "m42",     "displayName": "Orion Nebula",   "recommended": True},
    {"objectId": "ngc2237", "displayName": "Rosette Nebula", "recommended": False},
]
```

### Cell 2 — Imports & config

### Cell 3 — subType → type mapping
Starts with known mappings. If the data source returns an unmapped subType, the notebook raises an exception listing the unknown values so the mapping can be extended:
```python
SUBTYPE_TO_TYPE = {
    "Emission Nebula":   "nebula",
    "Reflection Nebula": "nebula",
    "Planetary Nebula":  "nebula",
    "Supernova Remnant": "nebula",
    "Dark Nebula":       "nebula",
    "Spiral Galaxy":     "galaxy",
    "Elliptical Galaxy": "galaxy",
    "Irregular Galaxy":  "galaxy",
    "Open Cluster":      "cluster",
    "Globular Cluster":  "cluster",
    "Double Star":       "star",
    # extended as new raw values are encountered
}
```

### Cell 4 — subType overrides
Flat `objectId → subType` dict for cases where the data source classification is wrong. `type` is always derived from the (possibly overridden) subType:
```python
overrides = {
    "ngc1499": "Emission Nebula",
}
```

### Cell 5 — Data source lookup
Query the chosen source(s) for each `objectId` to retrieve:
- `ra`, `dec`
- `constellation`
- `subType` (raw — will be mapped/overridden later)
- `magnitude`
- `angularSizeMajor`, `angularSizeMinor` (arcminutes)

Sources to evaluate (in priority order):
- **OpenNGC** (GitHub: mattiaverga/OpenNGC) — CSV, covers NGC+IC+Messier, has angular size, type, magnitude. Likely best single source.
- **astroquery.simbad** — reliable for RA/Dec, unreliable for type/size
- **astroquery.vizier** — access to many catalogs

Raw lookup results only in this cell — overrides are not applied here.

### Cell 6 — Build & validate DataFrame
1. Merge lookup results into DataFrame
2. Apply `overrides` — override values take precedence over looked-up values
3. Apply subType→type derivation via `SUBTYPE_TO_TYPE`
4. Validate:
   - No missing `ra`/`dec`
   - All subTypes are mapped — raise exception listing any unknown values
   - `objectId` is unique

### Cell 7 — Serialize
```python
df.to_json("dso.json", orient="records", indent=2)
```

---

## Serving
Commit `dso.json` to repo and enable GitHub Pages. Update `Config.DSO_URL` in AstroPlanner to point to the raw GitHub Pages file URL — replaces the current Cloud Function entirely. Nothing changes in the app except the URL — the Update screen, download flow, and JSON parsing all stay the same.
