# CLAUDE.md — HybridRenderingEngine

This file provides guidance for AI assistants working with the HybridRenderingEngine codebase.

---

## Project Overview

HybridRenderingEngine is an educational real-time 3D rendering engine built in C++/OpenGL 4.5. It implements **clustered forward shading** (a "hybrid" between forward and deferred rendering), combining GPU compute shaders for light culling with a forward physically-based rendering (PBR) pass.

**Key features:**
- Clustered shading: 16×9×24 = 3456 view-space clusters, max 1000 point lights
- PBR with Cook-Torrance BRDF (metallic-roughness workflow)
- Image-Based Lighting (IBL): diffuse irradiance + specular pre-filtering
- Shadow mapping: directional (2D) + omnidirectional point (cubemap)
- Bloom post-processing (extract → Gaussian blur → composite)
- HDR rendering with tone mapping and exposure control
- MSAA anti-aliasing

> **Note:** This project is no longer actively maintained (author: Angel Ortiz, now at Rockstar Games). It is preserved as an educational reference.

---

## Repository Structure

```
HybridRenderingEngine/
├── assets/
│   ├── models/                  # 3D assets (Sponza, DamagedHelmet — gLTF 2.0)
│   ├── scenes/                  # JSON scene definitions (sponza.json)
│   ├── shaders/
│   │   ├── *.vert / *.frag      # GLSL vertex and fragment shaders
│   │   ├── *.geom               # Geometry shaders (point shadows)
│   │   └── ComputeShaders/      # GPU compute shaders (clustering, light culling)
│   └── skyboxes/                # HDR equirectangular and cubemap skyboxes
├── include/                     # C++ header files (20 files)
├── src/                         # C++ implementation files (19 files)
├── libs/                        # Vendored third-party libraries
│   ├── assimp/                  # 3D model loading
│   ├── glad/                    # OpenGL function loader
│   ├── glm/                     # Math (vectors, matrices)
│   ├── imgui/                   # Immediate-mode debug GUI
│   ├── SDL2/                    # Window management and input
│   ├── nlohmann/                # JSON parsing
│   ├── gli/                     # DDS image loading
│   ├── KHR/                     # Khronos OpenGL extension headers
│   └── stb_image.h              # PNG/JPG/HDR image loading
├── modules/                     # CMake find_package modules
├── CMakeLists.txt               # Build configuration
└── README.md                    # Project documentation
```

---

## Building the Project

The project uses **CMake** (minimum 3.7) and targets **Windows** (primary). Linux/macOS are not officially supported.

### Requirements

- CMake 3.7+
- OpenGL 4.5+ capable GPU and drivers
- SDL2 (external install or bundled)
- ASSIMP (external install or bundled)
- OpenMP-capable compiler (MSVC, GCC, Clang)

### Build Steps

```bash
mkdir build
cd build
cmake ..
cmake --build .
```

The CMake configuration:
- Sets compiler flags per platform (MSVC / MinGW / Intel)
- Automatically copies `SDL2.dll` and `assimp-vc140-mt.dll` to the build directory (Windows post-build step)
- Sets the MSVC debugger working directory to `build/`

### Running

Execute `hybridRenderer` from the `build/` directory. The working directory must be `build/` because shader, model, and scene paths are relative to it.

---

## Architecture

### Startup Sequence

```
main.cpp
  └── Engine::startUp()
        ├── DisplayManager::startUp()   → SDL2 window + OpenGL 4.5 context + ImGui
        ├── SceneManager::startUp()     → Load default scene (sponza.json)
        ├── RenderManager::startUp()    → FBOs, shaders, SSBOs, IBL preprocessing
        └── InputManager::startUp()     → Attach to scene camera
```

### Main Loop

```
Engine::run()
  └── (each frame)
        ├── InputManager::processInput()   → keyboard/mouse → camera
        ├── SceneManager::update()         → scene state updates
        └── RenderManager::render()        → full render pipeline
```

### Manager Pattern

All subsystems follow the same lifecycle pattern — **no constructors do real work**:

```cpp
class FooManager {
public:
    bool startUp(/* dependencies */);
    void shutDown();
    // ... methods
};
```

Global instances are declared in `engine.cpp` and passed by pointer to subsystems that need them.

---

## Core Systems

### RenderManager (`include/renderManager.h`, `src/renderManager.cpp`)

Orchestrates the full render pipeline each frame:

1. **Shadow pass** — Render depth to shadow FBOs (directional + point lights)
2. **Cluster build** — Compute shader builds AABB per cluster in view space
3. **Light culling** — Compute shader assigns point lights to clusters
4. **PBR pass** — Forward shading with clustered light lookup (MSAA FBO)
5. **Resolve MSAA** — Blit MSAA buffer → resolve buffer
6. **Bloom** — Extract bright pixels → ping-pong Gaussian blur → composite
7. **Tone map** — HDR → LDR with exposure + Reinhard/ACES

Key uniforms and SSBOs set by `RenderManager` each frame:
- `ScreenToView` UBO — inverse projection, screen dimensions, cluster grid params
- `LightBuffer` SSBO — array of `GPULight` structs (binding 0)
- `LightIndicesBuffer` SSBO — light index list per cluster (binding 1)
- `LightGridBuffer` SSBO — offset+count per cluster (binding 2)
- `VolumeTileAABB` SSBO — cluster AABB data (binding 3)

### Shader System (`include/shader.h`, `src/shader.cpp`)

```cpp
Shader shader;
shader.init("path/to/vert.vert", "path/to/frag.frag");
shader.use();
shader.setInt("uTexture", 0);
shader.setMat4("uModel", modelMatrix);
```

`ComputeShader` is a lightweight struct for compute-only shaders:
```cpp
ComputeShader cs;
cs.init("path/to/shader.comp");
cs.use();
cs.dispatch(numGroupsX, numGroupsY, numGroupsZ);
```

### Scene Format (`assets/scenes/sponza.json`)

Scenes are defined in JSON:

```json
{
  "id": 0,
  "skyBox": { "folderPath": "...", "resolution": 512 },
  "models": [{ "path": "...", "scale": [...], "translate": [...] }],
  "camera": { "fov": 60, "near": 1.0, "far": 3000.0, "position": [...] },
  "dirLight": { "direction": [...], "distance": 1000, "shadowRes": 2048 },
  "pointLights": [{ "color": [...], "brightness": 1.0, "position": [...] }]
}
```

Parsed by `SceneManager` using nlohmann/json.

### Cluster Grid

- Grid: **16 (X) × 9 (Y) × 24 (Z)** = 3456 clusters
- Z-slices use **logarithmic depth** distribution
- Max lights: 1000 total, 50 per cluster
- SSBO data structures defined in `include/gpuData.h`:
  - `VolumeTileAABB` — min/max vec4 per cluster
  - `ScreenToView` — inverse projection + grid parameters for compute shaders

---

## Code Conventions

### Naming

| Element | Convention | Example |
|---|---|---|
| Classes/Structs | PascalCase | `RenderManager`, `GPULight` |
| Methods | camelCase | `startUp()`, `loadScene()` |
| Member variables | camelCase | `mWindow`, `numClusters` |
| Constants/macros | UPPER_SNAKE | `SCREEN_WIDTH`, `MAX_LIGHTS` |
| Global instances | `g` prefix | `gSceneManager`, `gEngine` |

### Coordinate Space Abbreviations (in comments)

| Abbreviation | Meaning |
|---|---|
| `mS` | Model space |
| `wS` | World space |
| `vS` | View space |
| `tS` | Tangent space |
| `sS` | Screen space |
| `lS` | Light space |

### File Header Comments

Every source file begins with:
```cpp
// AUTHOR: ...
// PROJECT: ...
// LICENSE: ...
// DATE: ...
// PURPOSE: ...
// SPECIAL NOTES: ...
```

Preserve this pattern when adding new files.

### Language Standard

C++14 with STL. Avoid C++17 features unless confirmed supported by the build environment. Heavy use of:
- `std::vector`, `std::string`, `std::unordered_map`
- GLM types (`glm::vec3`, `glm::mat4`, etc.)
- Raw OpenGL calls via GLAD

---

## Shaders

### GLSL Version

All shaders use `#version 430 core` or `#version 450 core`.

### Shader Files

| File | Type | Purpose |
|---|---|---|
| `PBRClusteredShader.vert/.frag` | Vert+Frag | Main PBR pass with clustered lights |
| `depthPassShader.vert/.frag` | Vert+Frag | Depth pre-pass |
| `shadowShader.vert/.frag` | Vert+Frag | Directional shadow map |
| `pointShadowShader.vert/.frag/.geom` | Vert+Frag+Geom | Omnidirectional point shadow |
| `skyboxShader.vert/.frag` | Vert+Frag | Skybox rendering |
| `screenShader.vert/.frag` | Vert+Frag | Fullscreen quad (tone map) |
| `blurShader.vert/.frag` | Vert+Frag | Gaussian blur (bloom) |
| `splitHighShader.vert/.frag` | Vert+Frag | Bloom bright-pass extraction |
| `brdfIntegralShader.vert/.frag` | Vert+Frag | BRDF LUT generation (IBL) |
| `buildCubeMapShader.vert/.frag` | Vert+Frag | Equirect → cubemap |
| `convolveCubemapShader.vert/.frag` | Vert+Frag | Diffuse irradiance convolution |
| `preFilteringShader.vert/.frag` | Vert+Frag | Specular pre-filter (IBL) |
| `ComputeShaders/clusterShader.comp` | Compute | Build cluster AABBs |
| `ComputeShaders/clusterCullLightShader.comp` | Compute | Assign lights to clusters |

### PBR Material Inputs (in `PBRClusteredShader.frag`)

The PBR shader samples these texture slots:
- `texture_albedo` — base color
- `texture_normal` — tangent-space normal map
- `texture_metallic` — metallic factor
- `texture_roughness` — roughness factor
- `texture_ao` — ambient occlusion
- `texture_emissive` — emissive color

---

## Known Limitations & TODOs

These are **intentional simplifications** for the educational scope — do not "fix" them unless explicitly requested:

1. **Static shadow maps** — Shadows do not update dynamically after the first frame.
2. **Frustum culling disabled** — Per-model AABB culling exists but is commented out pending a mesh system rewrite.
3. **Fixed screen resolution** — `SCREEN_WIDTH` / `SCREEN_HEIGHT` are compile-time constants (`1920×1080`).
4. **Single scene** — Scene switching is stubbed out; only `sponza.json` loads.
5. **No test suite** — There is no automated testing framework.
6. **Windows-primary** — CMake paths and DLL copying targets Windows; other platforms require manual adaptation.

---

## Making Changes

### Adding a New Shader

1. Add `.vert`/`.frag` (or `.comp`) files to `assets/shaders/`
2. Declare a `Shader` member in `renderManager.h`
3. Call `shader.init(...)` in `RenderManager::startUp()`
4. Call `shader.use()` and set uniforms in the appropriate render pass in `RenderManager::render()`

### Adding a New Light Type

1. Define a struct in `include/light.h` following the existing `GPULight` layout
2. Add it to the SSBO layout in `include/gpuData.h`
3. Update `clusterCullLightShader.comp` to include the new light in culling
4. Update `PBRClusteredShader.frag` to evaluate the new light contribution

### Adding a New Post-Process Effect

1. Create a new `FrameBuffer` variant in `include/frameBuffer.h` / `src/frameBuffer.cpp`
2. Add a new shader pair in `assets/shaders/`
3. Insert the pass between resolve and tone map in `RenderManager::render()`

### Modifying the Cluster Grid

Edit the constants in `include/renderManager.h`:
```cpp
const unsigned int gridSizeX = 16;
const unsigned int gridSizeY = 9;
const unsigned int gridSizeZ = 24;
const unsigned int maxLights = 1000;
const unsigned int maxLightsPerCluster = 50;
```
Also update the matching `#define` values in the compute shaders.

---

## Dependencies Quick Reference

All vendored libraries are in `libs/`. Do not modify them.

| Library | Version | Use |
|---|---|---|
| GLAD | — | OpenGL loader |
| GLM | 0.9.9+ | Math |
| SDL2 | — | Window + input |
| ASSIMP | — | Model loading |
| Dear ImGui | — | Debug UI |
| nlohmann/json | — | Scene JSON parsing |
| stb_image | — | PNG/JPG/HDR loading |
| GLI | — | DDS texture loading |

---

## Debugging Tips

- `debugUtils.h` provides `glCheckError()` — wrap suspect GL calls with it.
- ImGui debug windows are integrated; add panels in `RenderManager::renderGUI()`.
- OpenGL debug output is enabled at context creation (via `SDL_GL_SetAttribute` + `glDebugMessageCallback`).
- Cluster data can be visualized by outputting cluster index as a color in `PBRClusteredShader.frag`.
