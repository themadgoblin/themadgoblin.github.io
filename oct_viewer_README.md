# OCT File Viewer

A single-page web application that loads and renders `.oct` files using three.js.

## Usage

1. Open `oct_viewer.html` in a modern browser (Chrome, Firefox, Edge).
2. Select an `.oct` file using the **OCT** file input.
3. Select companion files using the **Buffers** input:
   - `.vbuf` / `.ibuf` — vertex and index buffer data.
   - `.mtb` — material bundle file (required for texture discovery in shipped builds).
4. Toggle **Load Textures** to enable texture prompting after load.
5. Click **Load**.
6. If textures were discovered, an interactive prompt appears in the log panel listing the required texture filenames. Click **Select .tbody Files** (or **Select Texture Files**) and browse to the matching files.

### Viewport Controls

- **Left-click drag** — orbit / rotate.
- **Scroll** — zoom in/out.
- **Right-click drag** — pan.

### Toolbar

| Button | Description |
|--------|-------------|
| **Wireframe** | Toggles wireframe rendering on all meshes. |
| **Export GLB** | Exports visible meshes as a `.glb` (glTF binary) file. |
| **Export OBJ** | Exports visible meshes as a Wavefront `.obj` file. |

### Mesh Visibility Panel

Above the log, a **Meshes** panel lists every loaded sub-mesh with a color swatch and triangle count. Click a row to toggle that mesh's visibility. Hidden meshes are excluded from exports.

## What It Parses

### OCT Binary Format

- **Header** (60 bytes): magic number (`0x45017629` LE / `0x29760145` BE), revision, string table length, atom count.
- **String Table**: null-terminated UTF-8 strings packed sequentially. String index size is 2 bytes if total strings <= 65536, otherwise 4 bytes.
- **Atom Data**: variable-length records encoding a tree of aggregates, scalars, and lists. Each atom has a 16-bit flags word encoding atom type, data type, length size, extra bits, and a parent depth index for tree reconstruction.
- **Tree Reconstruction**: uses the same stack-based algorithm as the engine's `ReadSerializer::BuildAtomHierarchy()` — compares each atom's `parentIndex` to the current depth to determine child/sibling relationships.

### Pools Extracted

| Pool | Key Fields Used |
|------|----------------|
| `VertexBufferPool` | `Name`, `Size`, `Data` (binary), `FileName` (external `.vbuf`) |
| `IndexBufferPool` | `Width` (bytes per index: 1/2/4), `Name`, `Size`, `Data`, `FileName` (external `.ibuf`) |
| `VertexStreamPool` | `Length`, `VertexBufferReference`, `VertexBufferOffset`, `ExtraStride`, `Elements` -> `Type`, `Name`, `Offset` |
| `IndexStreamPool` | `Length`, `IndexBufferReference`, `IndexBufferOffset`, `BaseVertexIndex`, `IndexStreamPrimitive` |
| `SceneTreeNodePool` | `Type` (`"Geometry"`), `NumPrimitives`, `Primitives` -> `Primitive` -> `IndexStreamReference`, `VertexStreamReferences`, `MaterialReference`, `UnitBase`, `UnitScale`, `Vdata`, `Idata` |
| `MaterialPool` | `FileName`, `Type`, nested `TextureReference` values |
| `TexturePool` | `FileName` |
| `MaterialBundlePool` | `FileName` (references the `.mtb` companion file) |

### Vertex Format Types Supported

| Type ID | Name | Size | Notes |
|---------|------|------|-------|
| 0-3 | `FLOAT_1` through `FLOAT_4` | 4-16 B | Standard floats |
| 24 | `U8_4` | 4 B | Bone indices |
| 25 | `U8_4_NORMALIZED` | 4 B | Vertex colors (RGBA) or bone weights |
| 30-31, 34-36 | `S10_10_10*` variants | 4 B | Packed normals/tangents |
| 39-40 | `S11_11_10*` variants | 4 B | Packed normals |
| 46 | `FLOAT16_2` | 4 B | Half-float UVs |
| 47 | `FLOAT16_4` | 8 B | Half-float vec4 |
| 48-51 | `S1_6_3/4`, `S3_12_2/3` | 3-6 B | Fixed-point formats |
| 54 | `U16_1` | 2 B | Single uint16 |
| 55 | `U8_1` | 1 B | Single uint8 |
| 56 | `UNIT_16` | 8 B | Compressed position (4x int16, decompressed via `UnitBase`/`UnitScale` AABB) |
| 57 | `FLOAT4x4` | 16 B | Matrix row |

### Vertex Semantics Recognized

`Position`, `Normal`, `Tangent`, `BaseMapUV`, `DetailMapUV`, `LightMapUV`, `Color`.

## Mesh Construction

The viewer uses multiple strategies to build meshes, tried in order:

### 1. Scene Tree Primitives (primary path)

Finds `Geometry` nodes in `SceneTreeNodePool` and reads their `Primitives` children. Two sub-paths exist:

- **Stream-pool path**: when `VertexStreamReferences` and `IndexStreamReference` are present, vertex/index data is read through the `VertexStreamPool`/`IndexStreamPool` indirection layer. Elements have explicit type and semantic annotations.
- **Inline Vdata/Idata path**: when stream pools are absent (common in shipped/platform builds), the `Vdata` and `Idata` integer lists encode buffer references, offsets, and strides directly. Vertex attribute layout is inferred by stride heuristics:
  - Stream 0 (stride >= 12): Position (`FLOAT32 x3`) at offset 0, UV (`FLOAT16 x2`) at offset 12 if stride >= 16, bone indices/weights at offset 16/20 for skinned meshes (stride 24).
  - Stream 1 (stride >= 8): Normal (`FLOAT16 x4`) at offset 0. Validated by checking that decoded normals have roughly unit length.

### 2. Heuristic Stream Pairing (fallback)

If no geometry nodes with primitives are found, pairs all available vertex streams with each index stream.

### 3. Brute-Force (last resort)

Tries every vertex stream (that has a `Position` element) against every index stream.

Triangle strips (`IndexStreamPrimitive = "trianglestrip"`) are converted to triangle lists. Degenerate and out-of-range indices are filtered.

## Texture Support

The viewer supports two texture discovery mechanisms:

### TexturePool-Based (older/unshipped builds)

When `TexturePool` has entries, materials reference textures via `TextureReference` indices in `MaterialPool` children. The viewer collects unique texture filenames and prompts the user to provide matching image files (PNG, JPG, TGA, etc.). Files are matched by base filename, ignoring extension.

### MTB-Based (shipped builds)

In shipped builds, `TexturePool` is typically empty. Instead:

1. The viewer reads `MaterialBundlePool` to find the `.mtb` filename.
2. If the `.mtb` was provided as a companion file, it parses the BNDL container:
   - **TEXB section**: extracts 8-byte texture hashes from 12-byte entries (4B flags + 8B hash).
   - Each hash corresponds to a `.tbody` file (e.g., `51ea68bd3ca35a3d.tbody`).
3. The texture prompt lists the `.tbody` filenames for the user to locate and select.
4. `.tbody` files are DDS-format compressed textures (BC1/BC3/etc.), loaded via Three.js `DDSLoader`.

### Fallback

If textures are not loaded or not found, a generic grey `MeshStandardMaterial` is used. Meshes with vertex colors use a white material with `vertexColors: true`.

## External File Conventions

| Extension | Description |
|-----------|-------------|
| `.vbuf` | Raw vertex buffer data, named `{octname}_{bufferIndex}.vbuf`. No header. |
| `.ibuf` | Raw index buffer data, named `{octname}_{bufferIndex}.ibuf`. No header. `Width` field determines bytes per index. |
| `.mtb` | Material bundle (BNDL container) with TEXB and MATP sections. Contains texture hashes and material parameters. |
| `.tbody` | Compiled texture body. DDS format, named by 8-byte content hash (e.g., `51ea68bd3ca35a3d.tbody`). |

Vertex/index data may also be embedded directly in the OCT as binary scalar atoms.

## Dependencies

Three.js r160 loaded from CDN (`cdn.jsdelivr.net`), including:
- `OrbitControls` — viewport navigation.
- `GLTFExporter` — GLB export.
- `OBJExporter` — OBJ export.
- `DDSLoader` — DDS/tbody compressed texture loading.

No build step or local dependencies required.
