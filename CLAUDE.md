# AstroPlannerData — Claude Working Notes

## Repo Purpose

Static file server for the AstroPlanner iOS/Android app. All files served from `mobile/` via GitHub Pages.

- `mobile/dso.json` — DSO catalog (`Config.DSO_URL`)
- `mobile/images.json` — Image catalog (`Config.IMAGES_URL`)
- `mobile/<objectId>.webp` — Resized astrophotography images

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

## image_processing.ipynb

Auto-discovers images, resizes to WEBP, generates `mobile/images.json`.

**Exports root:** `/Users/jwatts/Documents/astrophotography/Exports/`

Folder structure → objectId mapping:
- `Exports/DSO/<FolderName>/` → objectId = folder name lowercased
- `Exports/Planets/<Planet>/` → objectId = planet name lowercased (picks most recent session subfolder)
- `Exports/Lunar/` → always requires an `image_overrides` entry for `"moon"`

**Cell 1 (`image_overrides`):** Required for moon; optional for pinning thumbX/thumbY/thumbDim on any object.

**Workflow:** Add images to Exports → Run all cells → Commit `mobile/<objectId>.webp` and `mobile/images.json`

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
