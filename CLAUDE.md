# River Visualizations

A collection of WebGL-based visualizations exploring terrain generation and river-like particle flows. Built with Three.js.

## Project Overview

This repository contains two main types of prototypes:

**A. River Flow Visualizer** (`river-visualizer.html`) - Particle-based river simulation with DAG networks
**B. Landscape Visualizer** (`landscape-visualizer.html`) - Advanced terrain rendering with contours and metallic surfaces

Both share terrain generation capabilities but serve different purposes. Future work will explore integrating these approaches.

---

# A. River Flow Visualizer

Particle-based visualization of river flows across procedurally generated terrain.

## Concept

The visualizer renders tens of thousands of particle trails flowing along a user-defined river network (a Directed Acyclic Graph) that drapes over and curves through a 3D landscape. The goal is to create an organic, persuasive representation of water flowing across terrain.

### Key Design Principles

1. **Rivers as DAGs**: The river network is a directed acyclic graph where nodes are points on the terrain and edges are flow paths between them. Particles flow from upstream nodes to downstream nodes, splitting randomly at branches.

2. **Terrain Conformance**: Edges don't just connect nodes in straight lines—they "drape" over the terrain vertically (controlled by `give`) and curve laterally toward valleys (controlled by `lateralGravity`).

3. **Particle Width Illusion**: Individual particles are displaced laterally from the edge centerline to create the appearance of river width. The displacement varies along the edge (narrow at nodes, wide at mid-edge) to create natural pinching at confluences.

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        RiverVisualizer                          │
│  Main application class - orchestrates all components           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐ │
│  │  Heightmap  │  │ TerrainMesh │  │      RiverGraph         │ │
│  │             │  │             │  │                         │ │
│  │ - 2D height │  │ - Three.js  │  │ - Nodes (Map)           │ │
│  │   data      │  │   geometry  │  │ - Edges (Array)         │ │
│  │ - sampleH() │  │ - Custom    │  │ - DAG validation        │ │
│  │ - gradient  │  │   shader    │  │ - Path generation       │ │
│  └─────────────┘  └─────────────┘  └─────────────────────────┘ │
│         │                                      │                │
│         └──────────────┬───────────────────────┘                │
│                        ▼                                        │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │                    ParticleSystem                           ││
│  │                                                             ││
│  │  - 50k+ particles with trail history                        ││
│  │  - Edge assignment & progress tracking                      ││
│  │  - Lateral displacement (static offset + dynamic wobble)    ││
│  │  - THREE.LineSegments with additive blending                ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                 │
│  ┌─────────────────┐  ┌─────────────────┐                      │
│  │   NodeMarkers   │  │    EdgeLines    │                      │
│  │   (edit mode)   │  │  (debug viz)    │                      │
│  └─────────────────┘  └─────────────────┘                      │
└─────────────────────────────────────────────────────────────────┘
```

## Core Classes

### Heightmap
Generates and stores terrain elevation data using simplex noise with multiple octaves.

- `generate(scale, heightScale)` - Creates procedural terrain
- `sampleHeight(x, z)` - Bilinear interpolated height lookup
- `getGradient(x, z)` - Returns slope direction and magnitude for lateral gravity

### RiverGraph
Manages the DAG structure of nodes and edges.

- `addNode(x, z)` - Creates node snapped to terrain
- `addEdge(source, target)` - Creates edge with cycle detection
- `getValidEdges()` - Returns only edges that pass validation
- `getSourceEdges()` - Returns edges starting from source nodes (for particle respawn)

### RiverEdge
The most complex class—handles path generation with multiple processing stages:

1. **Initialize**: Straight line samples between source and target
2. **Relaxation**: Iteratively apply tension (smoothing) and gravity (downhill pull) forces
3. **Smoothing**: Laplacian filter passes to remove sharp kinks
4. **3D Conversion**: Sample terrain height, apply vertical draping
5. **Validation**: Mark edge invalid if too short, too steep, or problematic
6. **Lateral Vectors**: Compute perpendicular vectors for particle displacement

Key methods:
- `generatePath()` - Runs the full pipeline above
- `getDisplacedPointAt(t, lateralOffset)` - Returns 3D position with width modulation

### ParticleSystem
GPU-friendly particle management using typed arrays.

**State per particle:**
- `edgeIndices` - Which edge the particle is on
- `progress` - Position along edge (0-1)
- `speeds` - Individual speed variation
- `baseOffsets` - Static lateral offset (gaussian distribution)
- `wobblePhases` - Phase offset for sinusoidal wobble
- `trailPositions` - History buffer for trail rendering

**Key behaviors:**
- Trail reset on respawn (prevents teleport lines)
- Random branch selection at nodes
- Width modulation via `sin(t * π)` curve

## Configuration (CONFIG object)

### Terrain
- `size` - World units
- `segments` - Heightmap resolution
- `heightScale` - Vertical exaggeration
- `noiseScale` - Noise frequency

### River
- `give` - Vertical terrain conformance (0=straight, 1=drape)
- `lateralGravity` - Horizontal valley-seeking (0=straight, 1=strong)
- `pathSmoothing` - Laplacian smoothing strength
- `relaxationIterations` / `smoothingPasses` - Processing iterations
- `baseWidth` / `nodeWidth` / `midWidth` - Width parameters
- `wobbleAmount` / `wobbleSpeed` / `wobbleFrequency` - Animation params

### Particles
- `count` - Number of particles (50k default)
- `speed` - Flow rate
- `trailLength` - Trail segment count

## Edge Validity

Edges are marked `isValid = false` and excluded from particles/rendering if:
1. Horizontal distance < 0.3 units (too short)
2. Terrain penetration ratio > 0.8 with penetration > 0.2 (too steep)
3. Path length < 50% of horizontal distance (degenerate geometry)

## Interaction

- **E**: Toggle edit mode
- **Click** (edit mode): Add node, auto-connect to selected
- **Drag** (edit mode): Move node
- **Shift+Click**: Connect two nodes
- **Delete**: Remove selected node

## Rendering Details

- Particles use `THREE.LineSegments` with vertex colors
- Additive blending (`THREE.AdditiveBlending`) for glow effect
- `depthWrite: false` always, `depthTest` toggleable
- Terrain uses custom vertex/fragment shader with height-based coloring and grid overlay

## Performance Considerations

- All path computation happens at edge creation, not per-frame
- Particle update is O(n) with no allocations in hot loop
- Trail shift uses typed array index manipulation
- Wobble calculation skipped entirely when `wobbleAmount === 0`
- 50k particles with 8-segment trails = 350k line segments, runs smoothly on desktop

## Future Directions

Some ideas that would extend this naturally:
- Load heightmap from image (original design intent)
- River mesh rendering (not just particles)
- Velocity-based particle coloring
- Audio reactivity
- Export river network as JSON
- Touch/mobile support

---

# B. Landscape Visualizer

Advanced terrain visualization tool focused on high-quality rendering and topographic analysis. No particle systems - purely terrain-focused.

## Concept

A sophisticated terrain renderer with multiple visualization modes, contour generation, and metallic/crystalline surface effects. Designed for creating publication-quality terrain visualizations with fine control over appearance.

### Key Features

1. **Multiple Terrain Generation Modes**
   - Simplex Noise - Classic multi-octave noise
   - Sculpted - Advanced noise with domain warping, ridges, and valley bias
   - Image-based - Load heightmaps from bitmap images

2. **Color Plane Modes**
   - Height-based - Three-color gradient mapping by elevation
   - Slope-based (Normal Map) - Directional shading with rotation control
   - Iridescent - Animated rainbow shimmer effects

3. **Contour Lines**
   - D3.js-based contour generation with configurable interval
   - Terrain-colored or solid color modes
   - Shore lines at min/max elevation
   - Proper clipping and boundary line generation

4. **Surface Rendering**
   - Matt Surface - Standard terrain mesh with relief shading
   - Metallic Surface - Crystal-like shader with view-dependent effects
   - Point Grid - Sparse point cloud visualization
   - Wireframe mode

5. **Terrain Trimming**
   - Floor/Ceiling - Height-based terrain clamping
   - Directional Clipping - Crop terrain from cardinal edges (N/E/S/W)
   - Crisp boundary lines along clipped edges

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                   LandscapeVisualizer                        │
│  Main application class - orchestrates all components        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────┐  ┌──────────────┐  ┌─────────────────┐   │
│  │  Heightmap  │  │  ColorPlane  │  │  ContourLayer   │   │
│  │             │  │              │  │                 │   │
│  │ - Simplex   │  │ - Height map │  │ - D3 contours   │   │
│  │ - Sculpted  │  │ - Normal map │  │ - Line2 render  │   │
│  │ - Image     │  │ - Iridescent │  │ - Clipping      │   │
│  └─────────────┘  └──────────────┘  └─────────────────┘   │
│         │                 │                   │            │
│         └─────────────────┴───────────────────┘            │
│                           ▼                                │
│  ┌────────────────────────────────────────────────────────┐│
│  │               TerrainMesh                              ││
│  │  - PlaneGeometry with custom shader                   ││
│  │  - Wireframe/Point Grid modes                         ││
│  │  - Relief shading                                     ││
│  │  - Directional clipping (fragment discard)            ││
│  └────────────────────────────────────────────────────────┘│
│                                                             │
│  ┌────────────────────────────────────────────────────────┐│
│  │             MetallicSurface                            ││
│  │  - Separate mesh with crystal shader                  ││
│  │  - View-dependent effects (Fresnel, specular)         ││
│  │  - Edge detection and glow                            ││
│  │  - Optional iridescence                               ││
│  └────────────────────────────────────────────────────────┘│
│                                                             │
│  ┌──────────────┐  ┌────────────────┐                     │
│  │CameraControl │  │  ColorPlane    │                     │
│  │- Isometric   │  │  Texture Gen   │                     │
│  │- Perspective │  │  (Canvas-based)│                     │
│  └──────────────┘  └────────────────┘                     │
└─────────────────────────────────────────────────────────────┘
```

## Core Classes

### Heightmap
Multi-mode terrain generation with post-processing.

**Generation Modes:**
- `generateSimplex()` - Classic octave-based noise
- `generateSculpted()` - Advanced terrain with warping and ridges
- `loadFromImage()` - Import from bitmap heightmap

**Post-processing:**
- `applyLevels(floor, ceiling)` - Height clamping and normalization
- `sampleHeight(x, z)` - Bilinear interpolated lookup
- `toContourGrid()` - Export for D3 contour generation

**Sculpted Mode Parameters:**
- `heightScale` - Overall vertical scale
- `macroScale/macroStrength` - Large features (mountains, valleys)
- `warpStrength` - Domain warping intensity
- `ridgeMix` - Blend between smooth and ridged noise
- `valleyBias` - Push terrain toward valleys
- `detailScale/detailStrength` - Fine surface detail
- `roughness` - High-frequency bumpiness

### ColorPlane
Generates texture maps for terrain coloring using Canvas 2D API.

**Modes:**
- `generateHeightColors()` - Three-color gradient by elevation
- `generateNormalMap()` - Slope-based with rotation angle
- `generateIridescent()` - Animated procedural colors

**Methods:**
- `getTexture()` - Returns THREE.CanvasTexture
- `sampleColor(x, z)` - Get color at world position (for contours)

### ContourLayer
D3-based contour line generation with Three.js Line2 rendering.

**Features:**
- Uses `d3.contours()` for robust contour extraction
- LineMaterial for proper thickness in screen space
- Terrain-colored contours (samples ColorPlane)
- Automatic boundary line generation along clipped edges
- Line breaking at clipping boundaries (no stray lines)

**Key Methods:**
- `generate()` - Full contour generation pipeline
- `generateBoundaryLines()` - Crisp lines along trimmed edges

### TerrainMesh
Main terrain mesh with custom shader.

**Shader Features:**
- Fragment-based clipping (discard outside bounds)
- Relief shading (hillshade effect)
- Multiple render modes (plane/wireframe/points)
- Texture-based coloring from ColorPlane

**Clipping Implementation:**
Uses fragment discard based on world position:
```glsl
if (vPosition.z > (halfSize - clipNorth * terrainSize)) discard;
if (vPosition.z < -(halfSize - clipSouth * terrainSize)) discard;
if (vPosition.x > (halfSize - clipEast * terrainSize)) discard;
if (vPosition.x < -(halfSize - clipWest * terrainSize)) discard;
```

### MetallicSurface
Separate mesh layer with crystal/metallic shader effects.

**Shader Effects:**
- Fresnel rim lighting (view-angle dependent)
- Blinn-Phong specular highlights with controllable shininess
- Edge detection using `abs(dot(viewDir, normal))`
- Optional iridescence shimmer (HSV-based color shift)
- Opacity blending while maintaining depth write

**Parameters:**
- `opacity` - Blend between base terrain and metallic effect
- `baseBrightness` - Neutral terrain color level
- `shadowLift` - Ambient lighting (shadow brightness)
- `shininess` - Specular highlight sharpness
- `edgeSharpness` - Edge glow falloff
- `edgeBrightness` - Edge glow intensity
- `iridescence` - Rainbow shimmer strength

**Critical Design:** Material is always opaque (`transparent: false`) with shader-based opacity blending to maintain proper depth writing for occlusion.

### CameraController
Dual-camera system with smooth transitions.

**Modes:**
- Isometric (Orthographic) - For clean topographic views
- Perspective - For realistic 3D viewing

**Controls:**
- OrbitControls for manual interaction
- Auto-rotate and auto-pan options
- High-res mode (512 segments) for detailed terrain

## Configuration Structure

```javascript
CONFIG = {
  terrain: {
    mode: 'sculpted',           // 'simplex' | 'sculpted' | 'image'
    size: 20,
    segments: 256,              // 512 in high-res mode
    levelFloor: 0.15,           // Height trimming
    levelCeiling: 1.0,
    clipNorth: 0.01,            // Directional clipping
    clipEast: 0.01,
    clipSouth: 0.01,
    clipWest: 0.01,
    sculpted: { /* parameters */ }
  },
  color: {
    mode: 'height',             // 'height' | 'slope' | 'iridescent'
    lowColor: '#516a85',
    midColor: '#8b7355',
    highColor: '#e6c56b',
    slopeAngle: 0.0,
    iridescent: { /* parameters */ }
  },
  contours: {
    enabled: true,
    interval: 0.05,
    shoreLines: true,
    colorMode: 'terrain',       // 'terrain' | 'solid'
    opacity: 1.0,
    lineWidth: 1
  },
  surface: {
    enabled: false,             // Matt surface
    mode: 'plane',              // 'plane' | 'wireframe' | 'pointGrid'
    reliefShading: false,
    metallicShading: true,      // Metallic enabled by default
    metallic: {
      opacity: 0.15,
      baseBrightness: 1.0,
      shadowLift: 0.9,
      shininess: 2.8,
      edgeSharpness: 2.0,
      edgeBrightness: 1.0,
      iridescence: 0.0
    },
    opacity: 1.0
  },
  camera: {
    mode: 'isometric',
    autoRotate: false,
    autoPan: false
  }
}
```

## Rendering Details

**Contours:**
- Use Three.js Line2/LineMaterial for proper screen-space thickness
- Slightly thicker boundary lines (1.5x) for clipped edges
- Lines broken at clipping boundaries to prevent stray connections
- Positioned at y=0.02 to render above both surface types

**Surfaces:**
- Matt surface at y=0 (standard terrain mesh)
- Metallic surface at y=0.01 (slightly above)
- Both surfaces mutually exclusive (UI enforced)
- Both write depth for proper contour occlusion

**Clipping:**
- Fragment shader discard for surfaces
- JavaScript-based filtering for contours and point clouds
- Real-time updates without terrain regeneration
- Boundary lines automatically generated along clipped edges

## Performance Considerations

- Contour generation uses D3.js (can be slow at high resolution)
- High-res mode (512 segments) = 4x polygons but manageable at 60fps
- Metallic shader adds ~15-20 fragment instructions when enabled
- Point grid mode significantly reduces geometry
- All terrain generation cached - only regenerated on parameter change
- Clipping operates on GPU (fragment discard) for surfaces

## UI Organization

Collapsible sections in control panel:
1. **Terrain Generation** - Mode selection and parameters
2. **Terrain Trimming** - Floor/ceiling and directional clipping
3. **Colour Plane** - Color mode and palette
4. **Contours** - Interval, shore lines, styling
5. **Matt Surface** - Standard terrain rendering
6. **Metallic Surface** - Crystal shader effects
7. **Camera** - Projection and movement
8. **Scene** - Background color

All sections collapsed by default for clean initial state.

## Future Integration

Potential ways to combine with River Flow Visualizer:
- Particles flowing along contour lines
- River placement guided by valley detection in sculpted terrain
- Metallic shader applied to river surfaces
- Contours showing water depth or flow velocity
- Image-based heightmaps with pre-defined river networks
