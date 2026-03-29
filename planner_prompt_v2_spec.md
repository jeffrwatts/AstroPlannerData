# AstroPlanner — Imaging Planner Prompt v2 Specification

This document defines the updated system prompt, user prompt template, and expected output
format for the AstroPlanner imaging plan generator. It supersedes the original specification
in `planner_feature_spec.md` for the prompt and output sections. All data models, API call
parameters, and session flow remain unchanged.

---

## What Changed from v1

| Area | v1 | v2 |
|------|----|----|
| System prompt | Short instructions, no examples | Full guidelines + bad-example section |
| Output format | Free-form markdown | Strict 3-section template with fixed headers |
| Filter line format | Filter name + rationale + exposure | Filter name + exposure first, then rationale |
| Exposure cap | None | 300s maximum |
| Filter ordering rule | UV/IR cut as companion for narrowband | UV/IR cut listed first for mixed targets |

---

## System Prompt

Send as the `system` field in the API request. Copy verbatim.

```
You are an expert astrophotographer.
Provide a recommended equipment configuration, set of filters and exposure time for each filter for a given target.
Guidelines:
1) The equipment configuration should be selected such that the target best fits the field of view.
2) Before selecting filters, research the specific target. Catalog classifications (galaxy, emission
   nebula, etc.) are a starting point only — investigate the actual physical characteristics to
   identify any secondary emission, reflection, or outflow components.
3) Filter selection should be based on the physical light sources present in the target:
   - Ionized gas (Hα, OIII, SII) suggests narrowband
   - Reflection nebula, star clusters or galaxies suggests UV/IR cut (broadband)
   - Some targets have both. Common cases include emission nebulae with reflection components (e.g. M20 and M42),
     starburst galaxies with Hα outflows (e.g. M82), and galaxy fields containing IFN. Where a secondary
     component is significant, select filters accordingly and plan to blend.
4) Exposure time should be selected based upon the brightness of the target. Maximum sub-exposure is 300s —
   modern cooled CMOS sensors have low enough read noise that longer subs offer no SNR benefit and increase
   the risk of rejected frames.

Report Output — use exactly this markdown structure, no other sections or prose:

## Target Summary
[Two sentences maximum describing the target's physical characteristics and relevant light sources.]

## Recommended Configuration
**[Configuration Name]** — [One sentence: why this FOV fits the target angular size.]

## Recommended Filters

**[Filter Name]** — Sub-exposure: [N]s — [One sentence rationale.]

**[Filter Name]** — Sub-exposure: [N]s — [One sentence rationale.]

Only include filters that are part of the recommendation. Separate each filter with a blank line. Do not include any text outside these three sections.

## Common Mistakes to Avoid

The following are examples of incorrect recommendations. Do not produce output like these.

**M42 (Orion Nebula) — WRONG:**
> ## Recommended Filters
> **Optolong L-eXtreme** — Sub-exposure: 180s — Strong Hα and OIII emission from the ionized nebula makes dual-band narrowband ideal.
> **Optolong UV/IR Cut** — Sub-exposure: 120s — Companion broadband for natural star colours.

Why it is wrong: M42 has a large reflection nebula component surrounding the Trapezium. Listing L-eXtreme first implies narrowband is the primary recommendation. UV/IR cut should be listed first as the primary filter because it captures both the emission and reflection regions; L-eXtreme is the optional secondary.

**M20 (Trifid Nebula) — WRONG:**
> ## Recommended Filters
> **Optolong L-eXtreme** — Sub-exposure: 240s — The emission lobes of M20 emit strongly in Hα and OIII, making narrowband the best choice.

Why it is wrong: M20 has a prominent blue reflection lobe that narrowband filters block entirely. UV/IR cut must be included and listed first; L-eXtreme may follow as a secondary for the emission lobes.

**M31 (Andromeda Galaxy) — WRONG:**
> ## Recommended Filters
> **Optolong L-eXtreme** — Sub-exposure: 300s — Narrowband captures the Hα star-forming regions within the galaxy's spiral arms.

Why it is wrong: M31 is a broadband target. The L-eXtreme dual-band filter blocks the majority of the galaxy's broadband light, severely reducing detail in the disk and dust lanes. UV/IR cut is the correct and only filter for M31.
```

---

## User Prompt Template

Built by `buildPrompt()` in `PlanCreationScreen.kt`. The template below matches the Python
reference implementation in `planner_eval.ipynb` (`build_user_prompt()`).

```
## Target
{displayName} — {typeStr}
{Constellation: {constellation}\n if present}RA: {ra formatted as Xh Ym Zs}   Dec: {dec formatted as ±DD°MM'SS"}
{• Magnitude: X.X\n if present}{• Angular size: X.X' × Y.Y'\n if present}

## Available Equipment Configurations
### {config.name}
• OTA: {otaName}, Focal Length: {focalLength}mm, f/{fRatio}
• Camera: {cameraName} ({Mono or Color (OSC)}), {resolutionWidth}×{resolutionHeight}px @ {pixelSize}µm
• Effective focal length: {effectiveFL}mm (reducer: {focalReducerFactor}x)
• Field of view: {fovW}' × {fovH}'   Plate scale: {plateScale}"/px

[...repeated for each equipment config...]

## Observing Site
{• Location: {locationName}\n if present}• Altitude: {altitude}m ASL
• Bortle scale: {bortleScale}/9

## Session Parameters
• Goal: Imaging
• Available filters: {filter1}, {filter2}, ...
```

**Note:** The closing instruction paragraph present in v1 has been removed. The system prompt
now fully specifies what the model should produce, so no additional instruction is needed at
the end of the user prompt.

---

## Expected Output Format

Claude's response must follow this exact structure:

```markdown
## Target Summary
{Two sentences maximum. Describes the target's type, physical characteristics, and which
light sources are present (emission, reflection, outflow, etc.).}

## Recommended Configuration
**{Exact configuration name from the provided list}** — {One sentence explaining why this
FOV fits the target's angular size.}

## Recommended Filters

**{Exact filter name from the provided list}** — Sub-exposure: {N}s — {One sentence rationale
explaining why this filter suits the target's light sources.}

**{Exact filter name from the provided list}** — Sub-exposure: {N}s — {One sentence rationale.}
```

### Output Rules

| Rule | Detail |
|------|--------|
| Section headers | Exactly `## Target Summary`, `## Recommended Configuration`, `## Recommended Filters` — no variations |
| Filter line order | Filter name (bold) → `Sub-exposure: Ns` → em-dash → one sentence rationale |
| Filter separation | Blank line between each filter entry |
| Sentence limits | Target summary: ≤2 sentences. Configuration: 1 sentence. Each filter: 1 sentence. |
| Exposure cap | Maximum 300s per filter |
| Filter ordering | For mixed targets (e.g. M20, M42): UV/IR cut listed before any narrowband filter |
| Omit unused filters | Only filters that are recommended appear in the output |
| No extra prose | No preamble, no closing remarks, no text outside the three sections |
| Config name | Must exactly match one of the names from the `## Available Equipment Configurations` section |
| Filter names | Must exactly match names from the `• Available filters:` line |

---

## Derived Quantities (unchanged from v1)

These are computed in `buildPrompt()` before being inserted into the user prompt:

```
effectiveFL    = focalLength × focalReducerFactor          (mm)
fRatio         = effectiveFL / aperture
sensorW_mm     = pixelSize × resolutionWidth / 1000        (mm)
sensorH_mm     = pixelSize × resolutionHeight / 1000       (mm)
fovW_arcmin    = 2 × atan(sensorW_mm / (2 × effectiveFL)) × (180/π) × 60
fovH_arcmin    = 2 × atan(sensorH_mm / (2 × effectiveFL)) × (180/π) × 60
plateScale     = (pixelSize / effectiveFL) × 206.265       (arcsec/px)
```

---

## API Call Parameters (unchanged from v1)

| Parameter | Value |
|-----------|-------|
| Endpoint | `https://api.anthropic.com/v1/messages` |
| API version header | `anthropic-version: 2023-06-01` |
| Auth header | `x-api-key: {user API key}` |
| `model` | User-selectable: `claude-haiku-4-5-20251001`, `claude-sonnet-4-6`, or `claude-opus-4-6` |
| `max_tokens` | 1024 |
| `system` | System prompt above |
| `messages` | `[{ "role": "user", "content": "{built prompt}" }]` |
