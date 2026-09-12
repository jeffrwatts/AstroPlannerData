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

## aavso_report_check.ipynb

Compares an AAVSO Extended-format report (single comparison star + single check star, not an ensemble — see Q1 reasoning below) against the same session with a transform applied via VPhot's Transform Applier (`TRANS=YES`), to judge whether the transform improved accuracy/precision. There is no existing AAVSO tool for this; per Jeff's AAVSO mentor, most observers either skip this check entirely or do it ad-hoc in a spreadsheet — so this notebook is worth continued investment, not just a one-off script. Not committed to the repo output-wise — inputs are report `.txt` files from `../../../Documents/astrophotography/Photometry/...` (outside the repo).

**Inputs**, configured via `BEFORE_REPORT_PATH` / `AFTER_REPORT_PATH` / `GENERAL_REPORT_PATH`:
- `BEFORE_REPORT_PATH` — the untransformed AAVSO Extended report (required).
- `AFTER_REPORT_PATH` — the TA-transformed AAVSO Extended report. **Optional**: set to `None` for a session with no transform run yet (e.g. only reduced in one filter so far). All table-building/plotting functions take `df_after=None` and switch to single-report mode: one `Avg`/`Std`/`Trend` column set instead of paired `(Before)`/`(After)` columns, one curve per plot instead of two. The column-selection/formatting/styling code (in the Check Stars/Targets display block and `plot_before_after`) is written generically off of *which columns are actually present* so both modes share the same display logic — don't reintroduce Before/After-specific column names without also handling the single-report case.
- `GENERAL_REPORT_PATH` — VPhot's General Report for the (untransformed) session. Tab-delimited, **positional** columns (not name-keyed, since the target/check/comp column headers are the star names themselves): `JD, Filter, Airmass, <target>, <check>, <comp>, Err, SNR, FWHM, Skyglow, Max ADU, I.M.` `<comp>` is always `na` (comp star is the calibration anchor, no independent measured value). `Err`/`SNR` are per-observation and describe the **target's** measurement only (`Err ≈ 1.0857/SNR`) — there's no separate check-star SNR field. Only one General Report is needed (not a before/after pair): the transform recalibrates magnitude, it doesn't re-observe, so the same physical exposures/noise underlie both AAVSO reports.
- Joining the General Report onto the AAVSO report: the two exports compute timestamps independently and can disagree by a few `1e-5` days even for the same exposure, so an equality join on rounded JD silently drops rows. Use `attach_general_report()`'s nearest-JD-within-tolerance (`pd.merge_asof`, grouped by filter) instead.

**Two tables, back-to-back, no code cell in between (Jeff wants the display clean):**
- **Check Stars** — has a catalog magnitude (`KREFMAG`/`KREFERR` from the `NOTES` field), so accuracy is measurable. `Avg (Before/After)` shows `mag (Δ from catalog)`; Δ is color-coded green/orange/red vs. ±1×/2× `KREFERR`. `Std` is color-coded the same way against a `Noise Floor` column (per-filter median of the General Report's per-observation `Err`) — per the mentor, comparing check-star Std to the individual measurement's error estimate is a reasonable noise floor, and also a reasonable proxy for the target's own error, when check and target magnitudes are within ~1 mag (true for CY Aqr). `Trend (Before/After)` is the check star's predicted magnitude swing across the session's observed airmass range (linear fit of `KEFF` vs. `AMASS`), colored green/red at ±0.05 mag (mentor's rule of thumb for atmospheric extinction on long time series).
- **Targets** — no catalog magnitude for the target itself, so `Range` (observed Min–Max) is shown instead of Average; `Std` was removed as redundant once Range covers amplitude visually. For the V band only, `VSX Range (V)` shows the catalog amplitude fetched live from VSX (`fetch_vsx_range()`), and `Range (Before)`/`Range (After)` are colored via `range_color()`/`RANGE_TOLERANCE` (0.1 mag): good if the observed range falls fully inside the VSX range, borderline if it overshoots either edge by ≤0.1 mag, flag beyond that — per the mentor, don't fuss over VSX range differences under ~0.1 mag, since VSX's range is just observed min/max in the database, not authoritative. (Originally this was a strict inside/outside boolean; it flagged near-misses of a few hundredths of a mag as red on two different real sessions, which is why it's tolerance-aware now.) B/R rows stay uncolored (VSX only publishes one band's amplitude).

**Key constraint:** check-star rows must be merged Before-vs-After by `FILT` only, never by `KNAME` — a `TransformApplier` report renames the check star from a local designation (e.g. `117`) to an AAVSO AUID (e.g. `000-BCQ-590`) for the same physical star.

**Plots:** `plot_before_after()` shows check star and target curves side by side per filter (target curve added to support the "gut check on curve shape / outliers" criterion). `plot_diagnostics()` plots FWHM/Skyglow/Max ADU vs. time from the General Report, one subplot per metric, colored by filter — session-level (not before/after split), since both sides share the same exposures.

**Resolved:** an earlier revision left `Std` uncolored because this report format had no per-star SNR field to judge it against. The General Report closes that gap (see above) — the noise-floor coloring is no longer a rejected approach.

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
