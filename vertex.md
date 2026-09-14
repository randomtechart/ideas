# Vertex Animation Texture (VAT) Pipeline Plan
### Maya 2026 → Unity 6 URP
*Modeled on the OpenVAT approach, adapted for Maya export via Python/OpenMaya and Unity Shader Graph.*

---

## Overview

Core concept: bake per-vertex position (and optionally normal) for every animation frame into a 2D texture — width = vertex count, height = frame count — then decode it entirely in the vertex shader at runtime. This removes the animation from the CPU/skeleton entirely and moves it fully to the GPU.

- **Texture layout:** column = vertex index, row = frame.
- **Vertex order constraint:** vertex count and order must stay identical across every frame. Works for sims (nCloth, nHair, muscle, fluid-as-mesh), blendshapes, and baked skeletal deformation — not for topology changes.
- **Second UV set:** baked onto the export mesh, encoding `U = (vertexIndex + 0.5) / width` so the shader knows which texel belongs to which vertex.

---

## Part 1 — Maya-Side Export Tool (Python / OpenMaya)

### Inputs / UI
- Select mesh(es) to bake — support multi-mesh export in one pass (e.g. body + separate cloth mesh)
- Frame range (start/end), optional custom sample rate (e.g. bake every 2nd frame)
- Output folder + base filename
- Encoding mode toggle: **EXR float** vs **8/16-bit normalized PNG**
- Space toggle: **world space** vs **object space** (object space needed if repositioning the instance independently in Unity)
- Delta toggle: bake **absolute position** vs **delta from bind pose** (deltas compress better in normalized mode)
- Option to also bake normals (second texture)

### Pre-flight validation
- Confirm vertex count is constant across the frame range (sample frame 0 vs frame N via `MFnMesh` vertex count — cheap check before a full bake)
- Warn if history/deformers could rebuild topology and break vertex order
- Confirm texture width (vertex count) doesn't exceed platform max texture dimension (commonly 8192–16384) — flag if the mesh needs decimating or the bake needs splitting across multiple textures

### Bake pass
- For each frame: set `currentTime`, force DG evaluation, pull vertex positions via `OpenMaya.MFnMesh.getPoints()` (and `getNormals()` if enabled) into a preallocated NumPy array shaped `(frames, verts, 3)`
- Use Maya Python API 2.0 (`OpenMaya`) — not per-vertex `cmds.xform`, which is far too slow at scale
- World space → `MSpace.kWorld`; object space → `MSpace.kObject`

### Post-process
- Delta mode: subtract frame-0 positions from every frame
- Normalized encoding: compute global min/max per axis across all frames, remap array to 0–1
- Reshape array to `(height=frames, width=verts, channels)` and write:
  - **EXR path:** via `OpenImageIO` or `imageio`, 16-bit half float
  - **PNG path:** via `Pillow`/`imageio`, 16-bit PNG preferred over 8-bit if file size allows

### Mesh + UV export
- Duplicate the mesh at bind pose, create the second UV set (VAT UV) encoding vertex index on U
- Export as FBX with both UV sets intact

### Metadata sidecar (JSON)
- Vertex count, frame count, original fps, sample rate, space mode, delta mode, per-axis min/max (if normalized), texture filenames — read by the Unity-side importer/shader

### Packaging
- Wrap as a Maya shelf tool / `PySide2` or Maya-native UI window, callable via menu — reusable across projects rather than a one-off script

---

## Part 2 — Unity Import Settings

- Both textures (position + normal): **no compression**, **Linear color space** (not sRGB — this is vector data, not color), **Clamp** wrap mode, **no mipmaps**
- Filtering: **Point** on U (never blend between vertices), **Bilinear** on V only if frame-interpolation is desired

---

## Part 3 — Unity URP Shader (Shader Graph)

### Graph setup
- Target: URP **Lit** (or Unlit for full manual control)
- VAT decode only touches the **vertex stage** (Position + Normal overrides); fragment stage (base color, smoothness, normal maps, etc.) stays a normal Lit workflow

### Exposed properties
- `_PositionMap` (Texture2D)
- `_NormalMap` (Texture2D, optional)
- `_FrameCount` (Float)
- `_Frame` / `_Time0` + `_PlaybackSpeed` (Float) — driven by global time or per-instance offset
- `_BoundsMin`, `_BoundsMax` (Vector3) — only needed in normalized/PNG mode; omit for EXR mode
- `_Looping` (Boolean/Keyword) — frame wrap vs clamp

### Custom Function node — "SampleVAT"
Required because Shader Graph's built-in Sample Texture 2D node auto-picks mip level via screen-space derivatives, which don't exist in the vertex stage. Must use `SAMPLE_TEXTURE2D_LOD` explicitly inside a Custom Function node.

- Inputs: UV1 (the VAT UV set from Maya), texture, `_Frame`, `_FrameCount`
- Body:
  - `v = frac(_Frame / _FrameCount)` if looping, or `saturate(_Frame / _FrameCount)` if clamped
  - `SAMPLE_TEXTURE2D_LOD(_PositionMap, sampler_PositionMap, float2(IN.uv1.x, v), 0)`
  - Normalized mode: `lerp(_BoundsMin, _BoundsMax, sample.rgb)`; EXR mode: use `sample.rgb` directly
  - Output decoded `float3 position`

### Main graph flow
1. UV1 → Custom Function (position sample)
2. Result → **Add** with original `Position` (delta mode) or **Replace** entirely (absolute mode) → **Vertex Position** block
3. Same Custom Function pattern for `_NormalMap` if used → normalize → **Vertex Normal** block; otherwise pass authored normal through unchanged
4. Fragment stage unaffected — standard Lit shader graph downstream

### Per-instance variation (crowds)
- `_TimeOffset` exposed as a **per-instance** property so each instance's `_Frame` can offset independently without breaking GPU instancing/SRP batching
- At larger scale (`DrawMeshInstancedIndirect`), source the offset from a StructuredBuffer read in the Custom Function instead of a MaterialPropertyBlock

### Validation/debug aids
- Debug keyword to visualize raw VAT UV1 as color (confirms Maya-side UV bake correctness)
- Keyword to force nearest-frame sampling (isolates texture-side artifacts from shader-blend artifacts)

---

## Optimization Checklist (applies across the whole pipeline)
- Decimate mesh before baking — vertex count sets texture width directly
- Bake below gameplay framerate (e.g. 24 or 15 fps) and rely on bilinear V-axis filtering to interpolate — can halve texture height
- Verify first/last frame match for seamless loops
- Store deltas from bind pose rather than absolute position where possible — better precision utilization
- No texture compression on VAT textures (block compression destroys positional precision)
- Pair with LOD Groups using separate lower-vertex VAT bakes for distant instances
- Use `DrawMeshInstancedIndirect` / Entities Graphics + StructuredBuffer for large crowds to keep GPU instancing/SRP batching intact

---

## Open Next Steps
- Write the actual Maya Python/OpenMaya export script (standalone script vs. installable shelf tool/plugin — decide packaging first)
- Build the actual Shader Graph node-by-node wiring, or write the Custom Function HLSL bodies first
