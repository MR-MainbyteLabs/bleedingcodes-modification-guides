# PureTrace — Modification Guide
**Version:** 1.0.1  
**Package:** `src/puretrace/`  
**Entry point:** `puretrace-render` (CLI) or `python -m puretrace`  
**Dependencies:** stdlib only — `multiprocessing`, `hashlib`, `pickle`, `pathlib`, `math`, `struct`, `json`, `zlib`

---

## Who This Document Is For

Engineers and developers who want to add geometry types, extend the material system, change rendering settings, or embed PureTrace in a pipeline. This is not a usage guide. This document uses the actual class and function names from the source.

---

## Architecture Overview

```
cli.py          → entry point, argument parsing, scene selection, output writing
examples.py     → built-in scenes (functions returning Scene + Camera)
sceneio.py      → JSON scene loader
scene.py        → Scene — primitive list, BVH, light sampling, medium
geometry.py     → Sphere, Triangle, Quad, Primitive base class
materials.py    → PrincipledMaterial, BSDFSample
textures.py     → SolidColor, CheckerTexture, ImageTexture, EnvironmentMap, GradientTexture
bvh.py          → BVHNode — bounding volume hierarchy
integrator.py   → PathIntegrator — path tracing with MIS
renderer.py     → RenderConfig, Renderer, RenderState, tile dispatch
camera.py       → Camera — ray generation, depth of field
math3d.py       → Vec2, Vec3, Ray, ONB, RNG
output.py       → write_png, write_exr, read_exr, ACES tone mapper
rng.py          → RNG — PCG64 random, hemisphere sampling
```

---

## Public API (`__init__.py` exports)

```python
from puretrace import Camera, PrincipledMaterial, RenderConfig, Renderer, Scene, Vec2, Vec3
```

These are the only classes needed to build and render a scene.

---

## `RenderConfig` — Rendering Parameters

```python
@dataclass
class RenderConfig:
    width: int = 640
    height: int = 360
    samples_per_pixel: int = 128
    max_depth: int = 12                  # max ray bounces
    tile_size: int = 16                  # pixel tile size for parallel dispatch
    workers: int = 0                     # 0 = auto (os.cpu_count())
    samples_per_pass: int = 1            # samples per tile per worker pass
    seed: int = 1337
    russian_roulette_depth: int = 5      # start RR path termination at this depth
    sample_clamp: float = 0.0            # firefly clamp (0 = disabled)
    checkpoint_interval: float = 30.0   # seconds between checkpoint saves
```

### Key Tuning Parameters

**Noise vs render time:** `samples_per_pixel` is the primary quality lever. Doubling it halves noise (at twice the time).

**Render time:** `workers=0` uses all CPU cores. Set explicitly to limit parallelism: `workers=4`.

**Firefly suppression:** `sample_clamp` clamps each sample's radiance contribution to this value. `0.0` = disabled. Values of `4.0`–`10.0` suppress bright outliers at the cost of slightly reduced energy.

**Tile order:** Tiles are dispatched center-outward. The image center is visible first when writing preview images.

**Checkpoint interval:** `checkpoint_interval=30.0` saves a resume checkpoint every 30 seconds. Set to `0.0` to disable (no checkpointing).

---

## `PrincipledMaterial` — Material System

Disney-inspired metallic/roughness BSDF. All materials use this one class.

```python
@dataclass(slots=True)
class PrincipledMaterial:
    base_color: Texture | Vec3 | tuple = SolidColor(Vec3(0.8, 0.8, 0.8))
    metallic: float = 0.0           # 0 = dielectric, 1 = conductor
    roughness: float = 0.5          # 0.001 = mirror, 1.0 = diffuse
    transmission: float = 0.0       # 0 = opaque, 1 = fully transmissive (glass)
    ior: float = 1.5                # index of refraction (glass=1.5, diamond=2.4)
    emission: Texture | Vec3 | tuple = SolidColor(ZERO)
    emission_strength: float = 0.0  # > 0 makes this an emissive light source
    opacity: float = 1.0            # < 1 enables alpha transparency (null events)
    two_sided: bool = False          # emit from both sides of the surface
    name: str = "material"
```

### Material Presets

```python
# Mirror
PrincipledMaterial(base_color=Vec3(0.95, 0.95, 0.95), metallic=1.0, roughness=0.001)

# Glass
PrincipledMaterial(base_color=Vec3(1,1,1), transmission=1.0, roughness=0.001, ior=1.5)

# Rough gold
PrincipledMaterial(base_color=Vec3(1.0, 0.76, 0.33), metallic=1.0, roughness=0.3)

# Area light
PrincipledMaterial(emission=Vec3(1,1,1), emission_strength=15.0)

# Diffuse
PrincipledMaterial(base_color=Vec3(0.8, 0.2, 0.2), roughness=1.0)
```

### BSDF Implementation

The `sample()` method selects a lobe stochastically:

1. **Alpha transparency** (`opacity < 1.0`) — null event, rays pass through
2. **Transmission** — dielectric refraction/reflection via Fresnel; only `delta=True` events
3. **Specular** — GGX microfacet (probability weighted by Fresnel)
4. **Diffuse** — cosine-weighted hemisphere sampling

GGX is always `delta=False` even at very low roughness — only ideal dielectric and null events are marked delta for MIS weighting.

**To change roughness minimum** (currently 0.001, clamped in `__post_init__`):
```python
self.roughness = clamp(float(self.roughness), 0.0001, 1.0)
```

---

## Textures — `textures.py`

All textures implement `.value(uv: Vec2, point: Vec3) -> Vec3`.

| Class | What it does |
|---|---|
| `SolidColor(color)` | Constant color |
| `CheckerTexture(a, b, scale)` | Alternating checker pattern in UV or 3D space |
| `ImageTexture(path_or_array)` | Load PNG/PPM/EXR from disk or pass numpy array |
| `GradientTexture(a, b, axis)` | Linear gradient along X, Y, or Z axis |
| `EnvironmentMap(path)` | HDR environment (HDRI) for IBL; used by `Scene.environment` |

**To add a new procedural texture:**

```python
@dataclass(frozen=True, slots=True)
class NoiseTexture:
    scale: float = 1.0

    def value(self, uv: Vec2, point: Vec3) -> Vec3:
        # Simple hash-based noise
        n = math.sin(point.x * self.scale + point.y * 37.0 + point.z * 97.0) * 0.5 + 0.5
        return Vec3(n, n, n)
```

Pass it to `PrincipledMaterial(base_color=NoiseTexture(scale=5.0))`. The `as_texture()` helper in `materials.py` wraps raw `Vec3` and tuples into `SolidColor` — add `NoiseTexture` to the `isinstance` check there if you want it to pass through unchanged.

---

## Geometry — `geometry.py`

### Primitive Base

```python
class Primitive:
    material: PrincipledMaterial
    def hit(self, ray: Ray, t_min: float, t_max: float) -> HitRecord | None
    def bounding_box(self, t0: float, t1: float) -> AABB
    def surface_area(self) -> float
    def sample_surface(self, rng: RNG, time: float) -> SurfaceSample
```

### Built-in Primitives

**`Sphere(center, radius, material, motion_end=None)`**
`motion_end` enables motion blur — sphere linearly interpolates between `center` and `motion_end` at render time.

**`Triangle(v0, v1, v2, material, n0, n1, n2, uv0, uv1, uv2)`**
Normal and UV are per-vertex for smooth shading. If normals are not provided, geometric normal is used.

**`Quad(corner, u, v, material)`**
Parallelogram: `corner` is one corner, `u` and `v` are two edge vectors.

**Helper: `make_box(minimum, maximum, material) -> list[Quad]`**
Returns 6 quads forming an axis-aligned box.

**Helper: `lathe(profile_points, segments, material) -> list[Triangle]`**
Revolves a 2D profile around the Y axis to create a surface of revolution.

### Adding a New Primitive

Subclass `Primitive` and implement:
- `hit()` — ray-primitive intersection, returns `HitRecord | None`
- `bounding_box()` — returns `AABB` for BVH construction
- `surface_area()` — used for MIS light sampling PDF
- `sample_surface()` — uniformly sample a point on the surface for light sampling

Then use it in a scene:
```python
scene.add(MyPrimitive(...))
```

---

## Scene — `scene.py`

```python
scene = Scene()
scene.add(Sphere(...), Quad(...))   # add primitives
scene.environment = EnvironmentMap("hdri.hdr")   # optional IBL
scene.medium = HomogeneousMedium(density=0.05, albedo=Vec3(0.8,0.8,0.9), anisotropy=0.0)  # optional fog
scene.commit()   # builds BVH — required before rendering
```

`commit()` is called automatically by `Renderer.render()`. Only call it manually if you need to query the scene before rendering.

### Participating Medium — `HomogeneousMedium`

```python
@dataclass(frozen=True, slots=True)
class HomogeneousMedium:
    density: float        # scattering coefficient (higher = thicker fog)
    albedo: Vec3          # single-scatter albedo (0=absorb, 1=scatter)
    anisotropy: float     # Henyey-Greenstein g (-1=backward, 0=isotropic, 1=forward)
    max_distance: float = 100.0
```

---

## Camera — `camera.py`

```python
Camera(
    look_from=Vec3(0, 1, 3),
    look_at=Vec3(0, 0, 0),
    up=Vec3(0, 1, 0),
    vfov=45.0,              # vertical field of view in degrees
    aspect=16/9,
    aperture=0.0,           # 0 = pinhole (no depth of field)
    focus_distance=1.0,     # distance to focal plane (aperture > 0)
    shutter_open=0.0,       # motion blur start time
    shutter_close=0.0,      # motion blur end time (0 = no blur)
)
```

**Depth of field:** Set `aperture > 0.0` and `focus_distance` to the distance to your subject. Higher aperture = shallower depth of field.

**Motion blur:** Set `shutter_open=0.0` and `shutter_close=1.0` and give `Sphere` primitives a `motion_end` position.

---

## `Renderer.render()` — Main Render Call

```python
result = renderer.render(
    scene,
    camera,
    checkpoint="render.pkl",     # path to checkpoint file (None = disabled)
    resume=False,                 # True = resume from existing checkpoint
    preview="preview.png",       # write PNG preview after each pass (None = disabled)
    exposure=0.0,                 # EV exposure adjustment for PNG output
    tone_mapper="aces",          # "aces" | "reinhard" | "linear"
    progress=callback,           # optional: called with (fraction, avg_spp, elapsed_s)
)
```

**To add a progress callback:**
```python
def on_progress(fraction, avg_spp, elapsed):
    print(f"\r{fraction:.1%} ({avg_spp:.1f} spp, {elapsed:.0f}s)", end="", flush=True)

result = renderer.render(scene, camera, progress=on_progress)
```

**Tile dispatch:** Tiles are sorted center-outward before dispatch. To change to scanline order:
```python
dispatch_tiles = tiles   # remove the sorted() call in renderer.py
```

---

## Output — `output.py`

```python
write_png(path, width, height, pixels, exposure=0.0, tone_mapper="aces")
write_exr(path, width, height, pixels)   # 32-bit float HDR (no tone mapping)
```

`pixels` is `list[Vec3]` in row-major order (top-left origin).

`write_exr` requires no external libraries — it writes a minimal OpenEXR v1.x file using pure Python struct packing.

**Tone mappers:**
- `"aces"` — ACES filmic (default, good for photorealistic output)
- `"reinhard"` — simple Reinhard (brighter highlights, softer)
- `"linear"` — no tone mapping (gamma only)

**To add a new tone mapper:**

In `output.py`, find the `_tone_map()` function and add a new branch:
```python
elif tone_mapper == "neutral":
    # Neutral logarithmic tone mapper
    mapped = Vec3(
        math.log(1.0 + pixel.x * 4.0) / math.log(5.0),
        math.log(1.0 + pixel.y * 4.0) / math.log(5.0),
        math.log(1.0 + pixel.z * 4.0) / math.log(5.0),
    )
```

Then pass `tone_mapper="neutral"` to `renderer.render()` or `write_png()`.

---

## Built-in Scenes — `examples.py`

Functions that return `(scene, camera)`:

```python
from puretrace.examples import (
    cornell_box,
    material_showcase,
    checker_spheres,
    fog_demo,
    glass_spheres,
)
```

Each function accepts keyword arguments matching `RenderConfig` fields for convenience. Use these as templates for new scenes.

---

## Adding a Scene via JSON — `sceneio.py`

Scene JSON format:
```json
{
  "camera": { "look_from": [0,1,3], "look_at": [0,0,0], "vfov": 45 },
  "environment": { "type": "solid", "color": [0.5, 0.7, 1.0] },
  "objects": [
    {
      "type": "sphere",
      "center": [0, 0, 0], "radius": 1.0,
      "material": { "base_color": [0.8, 0.2, 0.2], "roughness": 0.5 }
    }
  ]
}
```

**To add a new object type to the JSON loader:**

In `sceneio.py`, find the dispatch block and add:
```python
elif obj_type == "my_primitive":
    primitives.append(MyPrimitive(...))
```

---

## Using PureTrace as a Library

```python
from puretrace import Camera, PrincipledMaterial, RenderConfig, Renderer, Scene, Vec3
from puretrace.geometry import Sphere, Quad, make_box
from puretrace.output import write_png

mat_floor = PrincipledMaterial(base_color=Vec3(0.8, 0.8, 0.8), roughness=0.9)
mat_light = PrincipledMaterial(emission=Vec3(1,1,1), emission_strength=10.0)

scene = Scene()
scene.add(*make_box(Vec3(-3,-0.01,-3), Vec3(3,0,3), mat_floor))
scene.add(Sphere(Vec3(0, 5, 0), 1.0, mat_light))

camera = Camera(look_from=Vec3(0,2,6), look_at=Vec3(0,0,0), vfov=40.0, aspect=16/9)

config = RenderConfig(width=1920, height=1080, samples_per_pixel=512, workers=0)
renderer = Renderer(config)
result = renderer.render(scene, camera, checkpoint="out.pkl", preview="out.png")

write_png("final.png", result.width, result.height, list(result.pixels))
```

---

## Common Modifications — Quick Reference

| What you want to do | Where to start |
|---|---|
| Change resolution or sample count | `RenderConfig` fields |
| Change max ray bounces | `RenderConfig.max_depth` |
| Change worker count | `RenderConfig.workers` |
| Enable firefly suppression | `RenderConfig.sample_clamp = 4.0` |
| Change tone mapper | `tone_mapper=` param in `renderer.render()` or `write_png()` |
| Add a new tone mapper | `output.py` → `_tone_map()` function |
| Create a mirror material | `PrincipledMaterial(metallic=1.0, roughness=0.001)` |
| Create a glass material | `PrincipledMaterial(transmission=1.0, ior=1.5, roughness=0.001)` |
| Create an area light | `PrincipledMaterial(emission=Vec3(1,1,1), emission_strength=N)` |
| Add a procedural texture | Implement `.value(uv, point) -> Vec3`, pass to `base_color=` |
| Add a new geometry type | Subclass `Primitive`, implement `hit`, `bounding_box`, `surface_area`, `sample_surface` |
| Add motion blur | `Sphere(center, r, mat, motion_end=Vec3(...))` + camera `shutter_close=1.0` |
| Enable depth of field | `Camera(aperture=0.1, focus_distance=N)` |
| Add a participating medium (fog) | `scene.medium = HomogeneousMedium(density=0.05, ...)` |
| Enable IBL (HDRI lighting) | `scene.environment = EnvironmentMap("hdri.hdr")` |
| Resume an interrupted render | `renderer.render(..., checkpoint="file.pkl", resume=True)` |
| Add a new JSON scene object type | `sceneio.py` dispatch block |
| Add progress reporting | `progress=callback` in `renderer.render()` |

---

*Built by MainbyteLabs — [github.com/MR-MainbyteLabs](https://github.com/MR-MainbyteLabs)*
