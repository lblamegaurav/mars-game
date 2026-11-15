# Mars Providence: Technical Roadmap, Architecture & Engineering Deep Dive

An exhaustive technical reference covering the entire architecture, subsystem engineering, multiplayer WebRTC protocol, performance optimization pillars, tech stack, and evolutionary roadmap of **Mars: Providence (Acidalia Recovery Protocol)**.

---

## 1. Executive Overview & System Vision

**Mars: Providence** is an open-world, high-performance WebGL 3D Martian rover expedition and survival simulation built to run seamlessly across high-end desktop rigs, Windows laptops with integrated/discrete GPUs, and low-power mobile devices. Players explore an infinite procedural Martian regolith landscape, locate scattered colonial transport survival modules using a directional radar, and collaborate in real-time with up to 6 players via peer-to-peer WebRTC data channels.

### Core Architectural Mandates
1. **Rock-Solid Framerate (60 to 144+ FPS Unlocked)**: Frame budgets strictly maintained under 8.3ms (for 120 FPS) to 16.6ms (for 60 FPS) with zero Garbage Collection pauses during gameplay.
2. **Zero-Asset Audio Synthesis**: 100% procedurally synthesized audio using the Web Audio API without multi-megabyte sound file overhead.
3. **P2P Multiplayer with Serverless Signaling**: Serverless WebRTC DataChannel star topology supporting up to 6 concurrent rovers with multi-region STUN traversal and late-join synchronization.
4. **Authentic Martian Visual Hierarchy**: ACESFilmic tone-mapped desert beige dunes, PCF shadow cascades, starry cosmic nebulae, and high-tech Lucida Calligraphy telemetry HUD.

---

## 2. Tech Stack & Specification Matrix

| Component | Technology | Version / Spec | Purpose & Architectural Justification |
| :--- | :--- | :--- | :--- |
| **Language & Platform** | JavaScript (ES2022+ Modules) | Modern ECMAScript | Native browser support, zero transpilation lag, modular ES imports. |
| **3D Rendering Engine** | [Three.js](https://threejs.org/) | `^0.187.0` | Production WebGL abstraction, scene graph, PBR materials, skeletal/instanced rendering. |
| **Asset Loader** | `GLTFLoader` | Three.js JSM | Fast parsing of binary `.glb` 3D meshes (Curiosity rover, artifacts, celestial skybox). |
| **P2P Networking** | [PeerJS](https://peerjs.com/) & WebRTC | `^1.5.5` (Bundled) | Low-latency WebRTC DataChannels, multi-region Google/Twilio STUN servers, star-relay. |
| **Procedural Audio** | Web Audio API | Native W3C Standard | Zero HTTP audio downloads; real-time procedural wind synthesis, engine pitch, and beacon chirps. |
| **Procedural Math** | Simplex Noise 2D | Custom implementation | Continuous C1-smooth Martian dunes with fractional Brownian motion. |
| **HUD & 2D Telemetry** | HTML5 Canvas 2D + Hardware CSS3 | Offscreen GPU Blitting | High-speed dials, oscilloscopes, and tactical compass tape without DOM thrashing. |
| **Build & Dev Toolchain**| [Vite](https://vite.dev/) | `v8.3.8` | Ultra-fast HMR (<15ms), tree-shaking Rollup bundler, asset compression. |
| **Deployment Target** | [Vercel](https://vercel.com/) | Edge Network CDN | Global static delivery, HTTP HEAD latency probe, automated GitHub CI/CD deployments. |

---

## 3. Subsystem Deep Dive & Inner Mechanics

### 3.1. Terrain Engine (`src/terrain.js`)
- **Infinite Procedural Chunk Streaming**: The world is divided into discrete square chunks (140m x 140m) configured on a 5x5 grid (700m x 700m active envelope) around the rover's position.
- **Dynamic Chunk Caching**: Chunks are stored in a `Map<string, ChunkData>` keyed by `"${cx},${cz}"`. When crossing chunk boundaries, out-of-range chunks are disposed (geometries freed from VRAM) and new chunks are allocated.
- **Elevation Formulation**: Simplex noise layered with multi-frequency fractional Brownian motion (fBm) combined with exponential crater depression fields.
- **Zero-Allocation Normal Computation**: `getNormal(x, z, target)` accepts a pre-allocated target vector, sampling elevation offsets to compute cross-product normal vectors without allocating temporary objects.
- **Natural Desert Regolith**: PBR material with custom canvas-generated diffuse and tangent normal textures calibrated to Martian beige (`#c5a682`), roughness 0.94, and zero metalness.

---

### 3.2. Rover Physics & Kinematic Rig (`src/rover.js`)
- **Ackerman Kinematics**: Longitudinal acceleration, deceleration, regenerative dynamic braking, and lerp-smoothed steering lock (+/- 0.55 rad).
- **Terrain Ground Clamping & Normal Orientation**: Samples terrain elevation directly beneath front and rear wheel axles; vehicle pitch and upright orientation align smoothly with terrain surface normals using quaternion slerp.
- **Headlights**: Dual downward-aimed warm spotlights (52.0 cd, angle 0.88 rad, penumbra 0.85) pooling light onto the sand. Shadow casting is explicitly disabled on headlights to eliminate redundant GPU shadow passes.
- **Wheel Dust Emitter**: Zero-allocation static particle pool emitting micro-dust puffs during wheel slip or acceleration.

---

### 3.3. Tactical Radar & Directional Sonar (`src/radar.js` & `src/ui.js`)
- **850-Meter Spatial Scan**: Computes relative polar coordinates of all uncollected survival modules relative to the rover's heading.
- **Orientation Normalization**: Targets ahead appear with an UP indicator; targets behind appear with a DOWN indicator.
- **Offscreen Dial Pre-Baking**: The radar scope's dark circular backing plate, concentric distance rings, crosshairs, and cardinal indicators are pre-baked onto an offscreen canvas (`sonarDialCanvas`). Per frame, the GPU performs a single `drawImage` blit, drawing only the dynamic rotating sweep needle and blip halos.

---

### 3.4. Multiplayer Subsystem & WebRTC Protocol (`src/multiplayer.js`)
- **Hybrid Star-Relay Architecture**: The room host acts as the authoritative star relay; clients establish direct WebRTC DataChannels to the host via PeerJS.
- **Multi-Region STUN Infrastructure**: Configured with Google (`stun.l.google.com:19302`) and Twilio public STUN servers with `iceCandidatePoolSize: 10`.
- **Normalized Room Discovery**: Codes entered as `4921`, `MARS-4921`, or `mars 4921` map to canonical peer identifier `mars-prov-mars4921`.
- **Dynamic Room Code Re-Assignment**: If an ID collision or timeout occurs on PeerJS signaling, a fresh code is generated and immediately propagated to the UI via `onRoomCodeChanged`.
- **In-Progress Game Synchronization**: When a player joins a game already underway, the host sends `auth_ok`, fires `start_game`, and serializes all collected item IDs via `{ type: 'sync_collected', ids: [...] }`.
- **Teammate Curiosity Rover GLB Clones**: Clones the parsed NASA Curiosity GLB model (`curiosity_rover.glb`) with team-specific emissive color accents, overhead callsign billboards, and glowing mast point lights with zero VRAM duplication.

---

### 3.5. Procedural Web Audio Engine (`src/audio.js`)
- **Zero Asset Overhead**: Zero external audio files downloaded for rover locomotion and ambient sound.
- **Martian Wind Simulation**: White noise routed into a lowpass `BiquadFilterNode` modulated via an LFO to simulate gusting winds.
- **Electric Traction Motor**: Dual oscillators frequency-modulated proportionally to rover velocity.
- **Proximity Beacon Chirps**: Dual-tone frequency-modulated sine burst triggered when closing within 75m of a salvage item.

---

## 4. The 5 Pillars of Performance Optimization

### Pillar 1: Zero-Allocation Render Loop
- Eliminated every `new Vector3()`, `new Matrix4()`, and array/object literal inside `requestAnimationFrame`.
- Reusable module-level scratch variables mutated in place via `.set()`, `.copy()`, and `.add()`.
- Bound instance method `this._animateBound` replaces per-frame anonymous arrow functions, eliminating Garbage Collection pauses.

### Pillar 2: Decoupled State & Offscreen Canvas Caching
- Speedometer dial plate and radar scope pre-rendered to offscreen canvases and blitted via a single GPU call.
- Oscilloscope waveform loops stepped at 2–3px intervals, reducing trigonometric calculations by ~70%.
- DOM HUD updates throttled to 10 Hz with strict dirty checking to prevent layout reflows.

### Pillar 3: GPU Fill-Rate & Hardware Context Harmonization
- Completely deleted the secondary camera feed, eliminating a secondary WebGL context, freeing GPU memory, and abolishing 6–10 auxiliary draw calls per frame.
- Context configured with `antialias: false`, `stencil: false`, and `powerPreference: 'high-performance'`.

### Pillar 4: Shadow Pass Decoupling
- Disabled spotlight shadow mapping on rover headlights, reducing shadow map rendering passes from 2 down to 1 per frame.

### Pillar 5: Clamped Delta Timing & Unlocked Framerate
- Delta time calculation unconstrained down to 0.001s and capped at 0.05s, natively supporting 60Hz, 75Hz, 90Hz, 120Hz, 144Hz, and 240Hz displays.
- Dynamic top FPS badge shifts color in Lucida Calligraphy (Green >= 50, Amber 30–49, Red < 30).

---

## 5. File System & Subsystem Mapping

```
spacegame/
├── index.html                   # HTML5 Entrypoint, HUD hierarchy, modal layouts
├── package.json                 # Project manifest (Three.js, PeerJS, Vite)
├── public/
│   ├── ping.txt                 # 2-byte static network probe endpoint
│   ├── models/
│   │   ├── curiosity_rover.glb  # NASA Curiosity rover 3D model (8.3MB)
│   │   ├── a_mountain.glb       # Distant horizon mountain massifs (12MB)
│   │   ├── alien_sky_equirect.webp # Starry celestial nebula skybox
│   │   ├── collectibles/        # 8 survival modules (GLB format)
│   │   └── spawn_props/         # Compressed non-collectible expedition props
│   └── videos/
│       └── intro.mp4            # Compressed cinematic landing sequence
└── src/
    ├── main.js                  # Engine entrypoint, camera rig, game loop, mode modals
    ├── terrain.js               # Infinite chunk streaming, Simplex noise, craters
    ├── rover.js                 # Kinematic chassis, suspension alignment, headlights
    ├── multiplayer.js           # WebRTC P2P star relay, Google STUN, GLB rover clones
    ├── radar.js                 # 850m spatial scan, relative Cartesian-to-polar tracking
    ├── items.js                 # Salvageable mission artifacts, pulsing beacon beams
    ├── weather.js               # Directional sunlight, PCF shadows, atmospheric fog
    ├── glbManager.js            # Asset pipeline, Curiosity model normalization
    ├── spawnProps.js            # Decorative expedition wreckage placement
    ├── audio.js                 # Procedural Web Audio API sound synthesis
    ├── ui.js                    # HUD orchestrator, offscreen dial caches, oscilloscopes
    └── style.css                # Visual design, Lucida Calligraphy typography, animations
```
