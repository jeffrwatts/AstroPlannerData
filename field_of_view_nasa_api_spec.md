# Field Of View Starfield Fetch — NASA SkyView API

How AstroPlanner's Field Of View viewer (`FieldOfViewScreen.kt`) fetches a DSS
starfield image for a target, so the same capability can be ported into the
`AstroPlannerData` notebook project (e.g. to render a finder chart for a
variable star's field).

## Service used

**NASA SkyView Virtual Observatory**, not a JSON API — a single HTTP `GET`
that returns image bytes directly.

```
https://skyview.gsfc.nasa.gov/current/cgi/runquery.pl
```

No API key is required.

## Request

Built in `FieldOfViewScreen.kt`:

```kotlin
private fun skyViewUrl(
    ra: Double, dec: Double, sizeDeg: Double, scaling: String, rotation: Float = 0f
) =
    "https://skyview.gsfc.nasa.gov/current/cgi/runquery.pl" +
    "?Position=$ra,$dec&Size=$sizeDeg&Pixels=1000&Rotation=$rotation" +
    "&Scaling=$scaling&Return=PNG&coordinates=J2000&Survey=DSS"
```

| Param | Value in AstroPlanner | Meaning |
|---|---|---|
| `Position` | `"$ra,$dec"` | Decimal degrees, J2000, e.g. `339.44937,1.53439`. SkyView also accepts a plain object name here instead of coordinates. |
| `Size` | field size in degrees | Width/height of the square cutout. See sizing below. |
| `Pixels` | `1000` (fixed) | Output image is always 1000×1000 px regardless of `Size`. |
| `Rotation` | degrees, `0`–`360` | Position-angle rotation of the returned image. `0` = north up. AstroPlanner lets the user rotate interactively; the rotation value sent is recomputed per request (see below). |
| `Scaling` | one of `Linear`, `Log`, `Sqrt`, `HistEq`, `LogLog` | Pixel-value transfer function, i.e. contrast stretch. User-selectable dropdown. |
| `Return` | `PNG` (fixed) | Response format. SkyView can also return FITS; PNG is simplest for direct display. |
| `coordinates` | `J2000` (fixed) | Epoch for the `Position` values. |
| `Survey` | `DSS` (fixed) | Digitized Sky Survey — the actual image data source. SkyView hosts many other surveys (2MASS, WISE, SDSS, etc.) selectable via this same param if broader wavelength coverage is ever wanted. |

The request is a plain `httpClient.get(url).readRawBytes()` (Ktor); the
response body is the raw PNG bytes, decoded directly by an image widget
(Coil's `AsyncImage` in the Compose app — a notebook would just do
`PIL.Image.open(io.BytesIO(response.content))` or similar).

## Field-of-view size calculation

`Size` isn't a fixed constant — it's derived from the imaging equipment so the
requested cutout roughly matches (with 10% margin) what the camera would
actually see:

```kotlin
private fun computeFov(config: EquipmentConfig): Pair<Double, Double> {
    val fl      = config.focalLength * config.focalReducerFactor
    val sensorW = config.pixelSize * config.resolutionWidth  / 1000.0  // µm → mm
    val sensorH = config.pixelSize * config.resolutionHeight / 1000.0
    val fovW    = 2.0 * atan(sensorW / (2.0 * fl)) * (180.0 / PI)
    val fovH    = 2.0 * atan(sensorH / (2.0 * fl)) * (180.0 / PI)
    return Pair(fovW, fovH)   // degrees
}
```

Then the requested `Size` is `max(fovW, fovH) * 1.1` — i.e. the larger of the
two sensor dimensions, padded 10%, used as a square cutout. The user can
override this via a slider (range 0.5°–5.0°) before re-fetching.

For a notebook without a specific camera config, just pick a reasonable fixed
`Size` (e.g. `1.0` to `2.0` degrees) or compute it the same way from
focal length / pixel size / resolution if those are known for the target
setup.

Plate scale (arcsec/pixel), shown as an info readout but not sent to the API:

```kotlin
private fun plateScale(config: EquipmentConfig): Double =
    (config.pixelSize / (config.focalLength * config.focalReducerFactor)) * 206.265
```

## Rotation and re-centering (interactive panning)

Only relevant if porting the interactive drag/rotate behavior — skip this
section for a simple static finder-chart fetch.

The app lets the user drag/rotate an overlay rectangle over the fetched
image, then "Update" re-fetches a new image centered/rotated to match. The
math:

- Pixel drag offset is un-rotated by the *previous* image's rotation to get
  a sky-frame (north-up) pixel offset, then converted to a degree offset
  using `displayedImageSize / canvasSize` as the deg/px scale.
- New Dec = `oldDec - dyDeg`; new RA = `oldRa - dxDeg / cos(newDec in radians)`
  (the `cos(dec)` term corrects for RA compression away from the equator).
- New rotation sent to SkyView = combination of the previous image's
  rotation and any additional two-finger rotation gesture, flip-adjusted
  and normalized to `[0, 360)`.

A finder-chart script generating one static image per target doesn't need
any of this — just pass the target's own RA/Dec and `Rotation=0`.

## Minimal Python-equivalent

```python
import requests
from PIL import Image
from io import BytesIO

def fetch_dss_image(ra_deg, dec_deg, size_deg=1.0, scaling="Linear", rotation=0):
    url = "https://skyview.gsfc.nasa.gov/current/cgi/runquery.pl"
    params = {
        "Position": f"{ra_deg},{dec_deg}",
        "Size": size_deg,
        "Pixels": 1000,
        "Rotation": rotation,
        "Scaling": scaling,
        "Return": "PNG",
        "coordinates": "J2000",
        "Survey": "DSS",
    }
    resp = requests.get(url, params=params, timeout=30)
    resp.raise_for_status()
    return Image.open(BytesIO(resp.content))
```

Note `ra_deg` must be **decimal degrees** (not hours) — same convention as
the app's internal DB storage, and the same conversion (`ra_hours * 15.0`)
needed if pulling RA from `vs.json`, which stores RA in decimal hours.
