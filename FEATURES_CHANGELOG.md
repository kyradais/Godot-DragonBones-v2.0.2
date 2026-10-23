# Added Features: Light Occlusion Support for DragonBones

## Overview
The recent branch (`Godot-DragonBones`) extends the original `Godot-DragonBones-Ori` by integrating **light occlusion** capabilities for `DragonBonesArmatureView`.  This allows the armature to behave as a true light occluder in `Godot 4.x`, enabling per‑node occlusion, shadow offsets, convex hull generation, and light mask control.

> This file is meant to serve as a running specification when creating new feature branches.

## Architecture Changes

- Introduced **`OccluderCullMode`** and **`OccluderShadowMode`** enums.
- Added new member variables to `DragonBonesArmatureView`:
  - `occluder_enabled`, `occluder_cull_mode`, `occluder_light_mask`,
    `occluder_polygon_mode`, `occluder_shadow_offset_x/y`, `occluder_shadow_mode`,
    `occluder_rid`, `occluder_polygon_rid`, and helper state flags.
- Implemented **convex‑hull** calculation (`compute_convex_hull`) using Andrew’s monotone chain algorithm with a fallback to raw points.
- Added `expand_polygon` to apply user‑supplied X/Y offsets to the generated polygon.
- Created helper `apply_occluder_shadow_layering` to update rendering server state when a hierarchy of light occluders is present.
- Updated `rendering_server` cleanup to free `occluder_rid` and `occluder_polygon_rid` correctly.
- Added `PackedVector2Array` inclusion for polygon handling.
- Updated class‑registration to expose the new properties under an **"Light Occluder"** group.

## New API (Godot Scriptable)

The following public methods and properties are now exposed on `DragonBonesArmatureView`:

| Property/Method | Type | Description |
|-----------------|------|-------------|
| `occluder_enabled` | bool | Toggle light occluder usage.
| `occluder_cull_mode` | enum (`Disabled`, `Clockwise`, `CounterClockwise`) | Cull mode for polygon.
| `occluder_light_mask` | int | Mask value of light sources that will see the occluder.
| `occluder_polygon_mode` | enum (`Raw Mesh`, `Convex Hull`) | Shape used for occlusion.
| `occluder_shadow_offset_x` | float | X offset applied to the shadow polygon.
| `occluder_shadow_offset_y` | float | Y offset applied to the shadow polygon.
| `occluder_shadow_mode` | enum (`Default`, `Full Bright`) | Shadow rendering quality.
| `apply_occluder_shadow_layering()` | void | Re‑apply shadow offsets when the node hierarchy changes.

> All properties are writable; the node will automatically register/unregister the occluder with Godot’s rendering backend.

## Usage Example (GDScript)
```gdscript
var view : DragonBonesArmatureView = get_node("ArmatureView")
view.occluder_enabled = true
view.occluder_cull_mode = DragonBonesArmatureView.OCCLUDER_CULL_CLOCKWISE
view.occluder_polygon_mode = 1  # Convex Hull
view.occluder_shadow_offset_x = 2.0
view.occluder_shadow_offset_y = 0.5
```

## Inspector Placement & Default Values

All new properties are registered under a dedicated **"Light Occluder"** group, so they appear collapsed in the **Inspector** panel of any `DragonBonesArmatureView` node, below the existing Animation / Debug Draw sections.

| Property | Group | Default | Value Type / Range |
|----------|-------|---------|--------------------|
| `occluder_enabled` | Light Occluder | `false` | Boolean |
| `occluder_cull_mode` | Light Occluder | `Disabled` (0) | Enum: `Disabled, Clockwise, CounterClockwise` |
| `occluder_light_mask` | Light Occluder | `1` (bit‑0) | Integer 0–1023 (10‑bit mask) |
| `occluder_polygon_mode` | Light Occluder | `Raw Mesh` (0) | Enum: `Raw Mesh, Convex Hull` |
| `occluder_shadow_offset_x` | Light Occluder | `0.0` | Float, applied to polygon X axis |
| `occluder_shadow_offset_y` | Light Occluder | `0.0` | Float, applied to polygon Y axis |
| `occluder_shadow_mode` | Light Occluder | `Default` (0) | Enum: `Default, Full Bright` |

Behavior per value:

- `occluder_cull_mode` — forwarded to `RenderingServer::canvas_occluder_polygon_set_cull_mode()`. `Disabled` lets the polygon cast shadow from both winding directions.
- `occluder_light_mask` — forwarded to `RenderingServer::canvas_light_occluder_set_light_mask()`. Only lights whose mask intersects this value see the occluder.
- `occluder_polygon_mode` — `Raw Mesh` uses the collected mesh outline verbatim; `Convex Hull` runs Andrew’s monotone chain (`compute_convex_hull`) and falls back to raw points when the hull is degenerate.
- `occluder_shadow_offset_x/y` — applied by `expand_polygon()` **after** hull computation, so the offset shifts the whole silhouette rather than individual mesh triangles.
- `occluder_shadow_mode` — `Default` keeps the character lit by scene lights; `Full Bright` sets light mask `0` and forces unshaded material so the character never receives an occluder shadow.

### GDScript Snippets

```gdscript
# Basic setup — raw mesh occluder affecting only light layer 1
@onready var view: DragonBonesArmatureView = $ArmatureView

func _ready() -> void:
    view.occluder_enabled = true
    view.occluder_cull_mode = DragonBonesArmatureView.OCCLUDER_CULL_CLOCKWISE
    view.occluder_light_mask = 0b0001
    view.occluder_polygon_mode = DragonBonesArmatureView.OCCLUDER_POLYGON_RAW_MESH
    view.occluder_shadow_mode = DragonBonesArmatureView.OCCLUDER_SHADOW_DEFAULT
```

```gdscript
# Convex hull silhouette with a small shadow offset, character stays unshaded
view.occluder_polygon_mode = DragonBonesArmatureView.OCCLUDER_POLYGON_CONVEX_HULL
view.occluder_shadow_offset_x = 2.0
view.occluder_shadow_offset_y = 0.5
view.occluder_shadow_mode = DragonBonesArmatureView.OCCLUDER_SHADOW_FULL_BRIGHT
```

```gdscript
# Re-apply layering after the node hierarchy changed
view.apply_occluder_shadow_layering()
```

```gdscript
# Toggle at runtime
view.occluder_enabled = false   # frees the occluder RID and detaches from canvas
view.occluder_enabled = true    # re-creates occluder + polygon RID on next draw
```

### C++ Registration Sketch

```cpp
ADD_GROUP("Light Occluder", "occluder_");
ADD_PROPERTY(PropertyInfo(Variant::BOOL, "occluder_enabled"),
        "set_occluder_enabled", "is_occluder_enabled");
ADD_PROPERTY(PropertyInfo(Variant::INT, "occluder_cull_mode", PROPERTY_HINT_ENUM,
        "Disabled,Clockwise,CounterClockwise"),
        "set_occluder_cull_mode", "get_occluder_cull_mode");
ADD_PROPERTY(PropertyInfo(Variant::INT, "occluder_light_mask", PROPERTY_HINT_RANGE, "0,1023,1"),
        "set_occluder_light_mask", "get_occluder_light_mask");
ADD_PROPERTY(PropertyInfo(Variant::INT, "occluder_polygon_mode", PROPERTY_HINT_ENUM,
        "Raw Mesh,Convex Hull"),
        "set_occluder_polygon_mode", "get_occluder_polygon_mode");
ADD_PROPERTY(PropertyInfo(Variant::FLOAT, "occluder_shadow_offset_x"),
        "set_occluder_shadow_offset_x", "get_occluder_shadow_offset_x");
ADD_PROPERTY(PropertyInfo(Variant::FLOAT, "occluder_shadow_offset_y"),
        "set_occluder_shadow_offset_y", "get_occluder_shadow_offset_y");
ADD_PROPERTY(PropertyInfo(Variant::INT, "occluder_shadow_mode", PROPERTY_HINT_ENUM,
        "Default,Full Bright"),
        "set_occluder_shadow_mode", "get_occluder_shadow_mode");
```

## Developer Notes

- The occlusion system now requires that the node be in the scene tree; calling `get_rendering_server()` is only valid during `_ready()` and subsequent lifecycle callbacks.
- Valid light masks are 10‑bit values; default is layer 1 (bit‑0).
- When `occluder_polygon_mode` is `Raw Mesh`, no convex hull is computed, and the poly is used verbatim.
- `apply_occluder_shadow_layering` must be called manually if you modify the node hierarchy after the occluder has been created.
- The `PackedVector2Array` format is used throughout; avoid using legacy `Array` for polygon data.
- Platform compatibility: This feature is available on all targets (Windows, Linux, macOS, Android, IOS, WASI). However, the `canvas_light_occluder_*` methods are only functional on 4.x and newer.

## Regression / Compatibility
- No breaking changes in the core API. Existing projects will not notice unless you start using the new properties.
- Existing code may see increased rendering server calls due to the added occluder cleanup.

## Next Steps
1. Add unit tests for occlusion logic.
2. Expose visual editor tools to edit the new properties, similar to how `light_mask` can be selected.
3. Optionally support occluder attachments to other node types.

---
This file was autogenerated by the PR that merged the new light‑occlusion features.
