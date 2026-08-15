# AstroPlannerData — Claude Working Notes

## Repo Purpose

Static file server for the AstroPlanner iOS/Android app, plus the notebooks that generate images for IslandSkiesWeb. Catalog JSON is served from `mobile/` via GitHub Pages; images and the image catalog are hosted on Cloudinary.

- `mobile/dso.json` — DSO catalog (`Config.DSO_URL`)
- `mobile/vs.json` — Variable star / standard field catalog
- Images and `images.json` — hosted on Cloudinary, produced by `image_processing_web.ipynb` (`Config.IMAGES_URL` points at the Cloudinary-hosted `images.json`, not this repo)

GitHub repo: https://github.com/jeffrwatts/AstroPlannerData

---

## Notebook Editing Rules

**Always re-read a notebook before editing.** Jeff edits notebooks directly in Jupyter between sessions, so the file on disk will often differ from the last known state.

Jupyter drops cell IDs on save. Before using any tool that targets cells by ID, reassign IDs first:

```python
import json
nb = json.load(open('dso_catalog.ipynb'))
for i, cell in enumerate(nb['cells']):
    cell['id'] = f'c{i:02d}'
with open('dso_catalog.ipynb', 'w') as f:
    json.dump(nb, f, indent=1)
```

---

## dso_catalog.ipynb

Generates `mobile/dso.json` from OpenNGC + SIMBAD fallback + astropy constellation lookup.

**Cell editing guide:**
| Cell | Purpose | Edit frequency |
|------|---------|----------------|
| Cell 1 (`objects` list) | Add new objects here — objectId, displayName, recommended | Every time a new object is added |
| Cell 4 (`SUBTYPE_OVERRIDES`) | Fix misclassified subTypes from data sources | Occasionally |
| Cell 5b (`MANUAL_DATA`) | Last-resort data for objects that can't be resolved | Rarely |

**Key design decisions:**
- RA is stored in decimal hours (degrees ÷ 15) to match the app's `DsoResponse` model
- Lookup order: OpenNGC → MANUAL_DATA → SIMBAD → not found
  - MANUAL_DATA is checked **before** SIMBAD so a partial SIMBAD result never overrides intentional manual data
- Missing constellations are filled automatically from RA/Dec using `astropy.coordinates.get_constellation` (IAU boundaries)
- `MANUAL_DATA` currently has entries for: `leotriplet`, `rhocomplex`

**Workflow:** Edit Cell 1 → Run all cells → Commit `mobile/dso.json`

---

## image_processing_web.ipynb

Auto-discovers images, resizes/re-compresses them, generates `web/images.json`, and uploads both images and the manifest to Cloudinary. This is the sole image pipeline — `image_processing.ipynb` (the old mobile-only GitHub Pages pipeline, `mobile/*.webp` + `mobile/images.json`) has been retired; AstroPlanner now fetches images and `images.json` from Cloudinary, same as IslandSkiesWeb.

**Exports root:** `/Users/jwatts/Documents/astrophotography/Exports/`

Folder structure → objectId/category mapping:
- `Exports/DSO/<FolderName>/` → objectId = folder name lowercased, category `"dso"`
- `Exports/Planet/`, `Exports/Comet/`, `Exports/Lunar/`, `Exports/Solar/` → category `"solar-system"`

**`UPLOAD_TO_CLOUDINARY`** (Cell 1): gates Cloudinary upload and constellation context chart generation — set `False` to just regenerate local files without touching Cloudinary.

**Workflow:** Add images to Exports → Run all cells → uploads happen inline (content-hashed, skips unchanged files) → commit `web/images.json` for reference.

See `specs/images_json_spec.md` for the full field reference.

---

## planner_eval.ipynb

Prompt evaluation framework for the AstroPlanner imaging planner feature. See `planner_prompt_v2_spec.md` for the full prompt specification.

- Uses `python-dotenv` — API key lives in `.env` (gitignored), never hardcoded
- `.env` format: `ANTHROPIC_API_KEY="sk-ant-..."`
- Test cases: M42, M20, M82, M31
- Custom graders: `grade_m20_filter_priority` (M42, M20), `grade_m82_narrowband_coverage` (M82)

---

## Git Conventions

- Commit messages describe the actual change, not generic summaries
  - Good: `"Add ic1396 as Emission Nebula"`
  - Bad: `"Update dso.json"`
- Always `git diff` before committing to write an accurate message
- `.env` is gitignored — verify with `grep -i "sk-ant" <notebook>` before committing notebooks

---

## User Notes

- Jeff works iteratively: runs cells, checks output, then asks for adjustments
- Prefers concise, direct responses
