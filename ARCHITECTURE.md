# Splottissimo Architecture

Splottissimo is a web-based procedural watercolor simulation built in a single HTML file using WebGL2. It simulates watercolor drops, stains, and solvent wipes on cold-press paper using a custom rendering pipeline.

## System Overview

The application is entirely client-side and relies on native WebGL2 via the HTML5 `<canvas>` API. There are no external dependencies or libraries. 

The architecture is split into two primary domains:
1. **Application Logic (JavaScript)**: Handles user input, timing, physical simulation state (drops, stains, rings), and scheduling.
2. **Rendering Pipeline (WebGL2 / GLSL)**: Handles the pixel-level visual representation of watercolor pigment, paper texture, fluid simulation logic, evaporation, and compositing.

## Application Logic

### State Management
- **Drops (`drops` array)**: Objects representing individual watercolor droplets or stains. They track position, radius, color, wetness, load, and internal physical parameters (frag, tendril, etc.). Drops start as "wet" and eventually "bake" into the canvas.
- **Queue (`queue` array)**: Handles delayed scheduling of drop spawns.
- **Rings (`rings` array)**: Objects representing concentric solvent wipes/blotters that lift pigment.
- **Theme (`THEME`)**: Controls the color palettes, color jittering, and palette harmonization to ensure splashes and stains sit within an aesthetically pleasing color family.

### Simulation Loop (`frame` function)
The simulation is driven by `requestAnimationFrame`. On each frame, the logic:
1. Processes the queue to spawn scheduled drops.
2. Randomly spawns automatic background drops, big stains, and solvent rings if timers allow.
3. Updates physical properties of active elements: drops accelerate/fall, hit the canvas, expand, and settle based on easing curves.
4. Categorizes drops into two lists:
   - **Wet**: Still actively spreading and need to be rendered dynamically.
   - **Bake**: Fully settled and ready to be stamped permanently into the background texture.

### Input
- **Pointer Down**: Triggers a `cluster` of small drops, a large `splash`, and a solvent `ring` at the interaction coordinates.
- **Keyboard**: 
  - `C`: Clears the paper and resets state.
  - `Space`: Pauses or resumes the simulation.

## Rendering Pipeline

The rendering pipeline heavily utilizes a "ping-pong" framebuffer technique to accumulate state over time. Two Framebuffer Objects (FBOs) with paired textures (`fb[0]/tex[0]` and `fb[1]/tex[1]`) alternate as source and destination.

### Shaders
The WebGL implementation is driven by a full-screen triangle vertex shader (`VERT`) and three primary fragment shaders:

1. **`FRAG_LIFT` (Evaporation & Solvent Wipe Pass)**
   - Computes evaporation (fading toward paper white) and the effect of solvent rings.
   - Solvent rings displace (smear) and remove (lift) pigment.
   - Writes updated accumulation and solvent coverage (stored in the alpha channel) to the destination FBO.

2. **`FRAG_BAKE` (Stamping Pass)**
   - Takes fully settled drops (those in the "bake" list) and permanently stamps them into the active FBO texture.
   - Uses multiplicative blending (`gl.blendFunc(gl.ZERO, gl.SRC_COLOR)`) and the Beer-Lambert law (`pigment()` function) to simulate physical ink absorption.
   
3. **`FRAG_MAIN` (Composite & Display Pass)**
   - Renders directly to the screen (null framebuffer).
   - Generates the cold-press paper texture (`paperColor`, `paperTooth`) combining procedural noise (FBM).
   - Samples the baked background texture (`sampleStain`).
   - Renders active, still-wet drops over the background.
   - Applies post-processing effects like vignetting and subtle film grain.

### Shader Uniform Optimization
Because GLSL loop limits and performance constrain the number of objects rendered per pass, data is packed tightly:
- `bufA`, `bufB`, `bufC`: Float32Arrays used to batch physical properties of drops into `vec4` uniform arrays (`uA`, `uB`, `uC`), capped at `MAXD` (32 drops per pass).
- `bufR`, `bufRB`: Batches properties of solvent rings into `vec4` uniform arrays, capped at `MAXR` (12 rings per pass).

## Procedural Generation (GLSL)
The visual look of the watercolor relies heavily on specialized GLSL functions:
- **`vnoise` / `fbm` / `fbm3`**: Value noise and Fractional Brownian Motion functions generate paper tooth, paper mottling, domain warping for lopsided silhouettes, and capillary tendrils.
- **`dropDensity`**: Calculates the physical footprint of a drop, incorporating parameters for squash (elliptical stretching), domain-warped edges, tendrils, and fragmentation (shards).
- **`ringLift`**: Calculates ragged, noisy concentric bands of solvent that wick outward, tearing and smearing the underlying pigment.
