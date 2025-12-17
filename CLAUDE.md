# River Flow Visualizer

A WebGL-based visualization of river-like particle flows across procedurally generated terrain. Built with Three.js.

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
