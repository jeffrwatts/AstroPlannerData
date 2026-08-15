# Field Check — Android Companion App Spec

## 1. Purpose

Native Android app that replaces the field-use half of `field_check.ipynb`. Goal: receive a plate-solved FITS frame shared directly from the ASI Air app, and within seconds show a go/no-go verdict on the target + AAVSO comparison stars — without connecting the ASI Air to a laptop or running ASTAP.

Companion to ASI Air, not a replacement for it. ASI Air still captures and plate-solves; this app only consumes the result.

## 2. User Flow

1. Jeff runs a first-light frame in ASI Air, confirms it plate-solved.
2. In ASI Air's image viewer, taps **Share → Share FITS**.
3. Android share sheet lists this app; Jeff picks it.
4. App opens directly to the analysis screen: parses the FITS, matches it to a catalog object, runs photometry, renders the annotated frame, shows GOOD/NEAR_EDGE/SATURATED/etc. per star and an overall verdict.
5. Jeff glances at it, decides go/no-go, gets back to the eyepiece.

No FITS file browsing, no manual object entry in the common case — the whole point is zero friction versus the current laptop+ASTAP workflow.

## 3. Non-Goals

- No plate solving (ASI Air/ASTAP already did this — we only read the WCS it wrote).
- No live stacking, guiding, or sequence control.
- No general-purpose FITS viewer (no arbitrary header browsing, no multi-extension support).
- No catalog editing on-device — `vs.json` / comp-star cache stay authored by the existing notebooks and published via GitHub Pages; the app only reads them.

## 4. Inputs and Assumptions

- **Trigger**: `Intent.ACTION_SEND` with a FITS payload as a `content://` URI, from ASI Air's "Share FITS" action.
  - **Open risk**: the exact MIME type ASI Air's share sheet uses is unverified (likely `application/octet-stream` or `*/*`, since FITS has no standard Android MIME registration). Manifest intent filter should accept `*/*` for `ACTION_SEND` and sniff by filename extension (`.fit`, `.fits`) as a fallback, rather than gating on a specific MIME type. This must be validated against a real ASI Air share in Phase 1 before anything else is built on top of it.
  - Also register `ACTION_VIEW` for `.fits`/`.fit` as a bonus entry point (opening from a file manager), since it's nearly free once `ACTION_SEND` handling exists.
- **FITS content**: single-HDU 16-bit unsigned mono Bayer frame (matches `field_check.ipynb` Cell 3 assumptions) with a `BAYERPAT` header keyword, and a WCS solution written by ASTAP/ASI Air — `CTYPE1/2 = RA---TAN/DEC--TAN`, `CRVAL1/2`, `CRPIX1/2`, and a CD matrix (`CD1_1`/`CD1_2`/`CD2_1`/`CD2_2`) or the `CDELT`+`CROTA2` fallback form.
- **Target identification**: the FITS `OBJECT` header keyword (set from the ASI Air sequence/plan name) is matched case-insensitively against `vs.json` `objectId`/`displayName`. If no match, fall back to a manual picker (searchable list of catalog objects) rather than blocking — Jeff can always tell the app what he's imaging.
- **Network**: assume little-to-no internet in the field (phone likely joined to the ASI Air's own WiFi AP). Catalog data (`vs.json` + `mobile/comp_stars/*.json`) must be cached locally ahead of time and the analysis pipeline must run fully offline, same split as the notebook's catalog-build-time vs. field-use-time division.
- **`vs.json` `type` field**: every entry has `type` = `"variable"` (an AAVSO/VSX variable star) or `"standard_field"` (an AAVSO photometric standard field, e.g. `SA110` — a dense sequence of calibration stars, imaged deliberately defocused to spread star light across the Bayer matrix for color-transform calculations, not to monitor variability). See `specs/vs_json_spec.md` for the full field reference. Practical effect on this app:
  - For `standard_field` targets, `subType`/`magnitude`/`variablePeriodDays`/`variableEpochJd`/`variableEpochType` are all `null` — don't attempt to show or compute anything variability-related (no phase, no min/max countdown) for these targets.
  - The manual object picker and any target-type label in the UI should distinguish the two (e.g. "Standard Field" vs. the star's `subType`) so Jeff isn't confused seeing null variability fields for a target he intentionally defocused.
  - The corresponding `comp_stars/<objectId>.json` cache entries for a `standard_field` target carry a `bands` field (per-star multi-band magnitudes) that ordinary variable-star comparison stars don't have — see `specs/comp_stars_json_spec.md`. Not required for the go/no-go photometry pipeline (which only needs `ra`/`dec` to place apertures), but worth surfacing in the per-star results table if there's room, since color-transform work is the whole point of a standard-field frame.
  - Comparison-star counts differ sharply by design: a `standard_field` cache can have up to 30 stars (capped, brightest-first) in a native FOV as small as ~15′, versus a handful of stars for an ordinary variable across a much wider generic window — expect (and don't warn on) a denser annotated frame for standard-field targets.

## 5. Architecture

Single-module Android app is fine at this scale; internal packages mirror the notebook's cell boundaries so the mapping stays obvious:

```
app/
  ui/                 Compose screens: ShareReceiverActivity, AnalysisScreen, ObjectPickerScreen, SettingsScreen
  fits/               FitsReader — header + 16-bit image data parsing
  wcs/                WcsSolver — TAN projection pixel<->sky
  photometry/         StarSampler (centroid, aperture peak/background), GaussianFwhmFitter
  catalog/            CatalogRepository — vs.json + comp_stars cache, sync-when-online, object matching
  render/             BayerDebayer, PreviewStretch, FrameAnnotator (Canvas overlay drawing)
  model/              Star, StarResult, FrameAnalysis, StarStatus enum
```

### Data flow (mirrors notebook cells 3 → 7)

```
ACTION_SEND (FITS content:// Uri)
        │
        ▼
FitsReader.parse(uri)  ──────────────►  header, raw16 (ShortArray/IntArray), width, height, bayerPattern
        │
        ▼
WcsSolver.fromHeader(header)  ───────►  pixel<->sky transform (Cell 3)
        │
        ▼
CatalogRepository.resolveTarget(header, wcs)
        │  - match OBJECT header → vs.json entry (or manual picker)
        │  - load cached comp_stars/<objectId>.json
        ▼
List<Star> (target + comps, RA/Dec)  ──────────────►  (Cell 4)
        │
        ▼
for each star: WcsSolver.worldToPixel → StarSampler.sample(raw16, px, py)
        │  - centroid refine, peak ADU, local background, GaussianFwhmFitter
        │  - classify: OUT_OF_FRAME / SATURATED / TOO_FAINT / NEAR_EDGE / UNDERSAMPLED / GOOD
        ▼
List<StarResult>  ─────────────────────────────────►  (Cell 5)
        │
        ▼
BayerDebayer.toRgb(raw16, bayerPattern) → PreviewStretch.asinhZscale(rgb)  ─►  (Cell 6)
        │
        ▼
FrameAnnotator.draw(bitmap, results)  ───────────────────────────────────►  (Cell 7)
        │
        ▼
AnalysisScreen: annotated Bitmap + per-star status table + overall verdict banner
```

## 6. Key Algorithm Notes (porting from Python)

- **WCS (TAN projection)**: no astropy on Android, but the math is closed-form — standard gnomonic (TAN) forward/inverse transform using `CRVAL`, `CRPIX`, and the CD matrix (or `CDELT`×rotation-matrix-from-`CROTA2` if CD keywords are absent). This is a well-documented ~30-line implementation, no library needed. Only TAN is in scope — that's what ASTAP/ASI Air write.
- **FITS parsing**: header is fixed 80-byte ASCII cards in 2880-byte blocks — trivial to hand-parse for the handful of keywords needed (`NAXIS1/2`, `BITPIX`, `BSCALE`/`BZERO`, `BAYERPAT`, `OBJECT`, WCS keywords). Data block is big-endian 16-bit (typically `BITPIX=16` with `BZERO=32768` for unsigned). Recommend a small custom parser over pulling in `nom-tam-fits` — our needed keyword set is narrow and a custom parser avoids an unvetted dependency's Android compatibility risk.
- **Centroid + aperture photometry**: direct port of `sample_star()` (Cell 5) — local background via median of a cutout, centroid-of-mass refine, peak ADU and background ADU from small apertures. Pure arithmetic, no library needed.
- **2D Gaussian FWHM fit**: `astropy.modeling` + `LevMarLSQFitter` has no Android equivalent, but fitting amplitude/x0/y0/σx/σy to a ~30×30px cutout is a small enough problem for a hand-rolled Gauss-Newton or Levenberg-Marquardt loop (5 params, few dozen iterations, small residual vector). No external solver library needed.
- **Bayer debayer + stretch**: this is preview-only (never feeds photometry, which always samples `raw16` directly — same separation as Cells 5 vs. 6 in the notebook). A simple 2×2-block bilinear or even nearest-neighbor debayer is sufficient for a go/no-go preview; full `cv2`-quality demosaicing is unnecessary. ZScale + asinh stretch is a few dozen lines of arithmetic, no library needed.
- **Net result**: no OpenCV, no astropy-equivalent, no FITS library dependency required. Everything ports to plain Kotlin arithmetic. This keeps the app small and avoids chasing Android compatibility of scientific-Python-derived libraries.

## 7. Catalog Sync

Reuse the existing repo's publish pattern: `vs.json` and `mobile/comp_stars/*.json` are already static files on GitHub Pages (`Config.VS_URL`-equivalent). `CatalogRepository`:

- On app launch (or explicit "Refresh Catalog" action) with connectivity: fetch latest `vs.json` + all `comp_stars/*.json`, write to app-private storage.
- Analysis always reads from the local cache, never touches network mid-analysis — matches `field_check.ipynb`'s "zero network calls in the field" rule from [[vs_catalog_comp_stars]].
- First-run with no cache and no connectivity: show a clear "no catalog cached yet — connect to WiFi once before your next session" state rather than failing silently.

## 8. Config / Thresholds

Port `field_check.ipynb` Cell 1 constants as in-app defaults, editable from a Settings screen rather than hardcoded (unlike the notebook, there's no "just edit the cell" escape hatch on a phone):

`SATURATION_ADU`, `MIN_GOOD_ADU`, `EDGE_MARGIN_PX`, `CENTROID_SEARCH_PX`, `PHOTOMETRY_RADIUS_PX`, `MIN_FWHM_PX`, `STRETCH_A`.

## 9. UI

Single analysis screen:
- Top: overall verdict banner (e.g. "GOOD — target + 4/6 comps usable" in green, or a red/yellow warning summarizing what's wrong).
- Center: annotated frame (pinch-zoom/pan), same visual language as notebook Cell 7 — status-colored circles, target always red, labels only (no ADU/FWHM clutter on-image).
- Below/behind a toggle: per-star results table (label, role, peak ADU, background, FWHM, status) — the Cell 5 table.
- Manual object picker, reachable if auto-match fails or picks the wrong object.

## 10. Phased Build Plan

1. **Share intake + FITS parsing**: verify real ASI Air "Share FITS" intent shape on a device; header/data parser; dump parsed header + raw stats to a plain screen. Validates the biggest unknown first.
2. **WCS + catalog + photometry**: TAN projection, catalog repository + sync, object matching, `sample_star` port, status classification — table-only UI (no image yet).
3. **Render + annotate**: debayer, stretch, `FrameAnnotator` overlay — the full visual.
4. **Polish**: settings screen for thresholds, manual object picker UX, offline-cache empty-state handling, app icon/branding.

## 11. Open Questions Before Coding Starts

- Confirm real MIME type / extras on the Intent ASI Air's "Share FITS" produces (needs a live test — can't be determined from this repo).
- Confirm the `OBJECT` header value ASI Air writes actually matches (or can be reliably mapped to) `vs.json` `objectId`/`displayName` for Jeff's current targets (CY Aqr, YZ Boo, W UMa).
- Min supported Android version / target device (affects Compose version, storage APIs for the shared URI).
