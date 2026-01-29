# Regolith Bin Terrain Generator (PyDEME / DEME)

This script generates a polydisperse regolith bed inside a rectangular “bin” using DEME (PyDEME). It spawns rigid **clumps** (each clump = one regolith “grain” built from multiple spheres), lets each batch settle under gravity, and writes out particle states plus final clump and contact data.

## What it produces
- A sequence of per-frame CSV files: `regolith_bin_output_0000.csv`, `regolith_bin_output_0001.csv`, ...
  - Each file contains particle positions (`XYZ`) for that frame.
- `clumps.csv`
  - Final clump states (grain positions/orientations etc., depending on DEME’s clump CSV format).
- `contact_pairs.csv`
  - Final **contact pairs**: which bodies are touching at the output time (used to compute contact forces).

All outputs go to `OUT_DIR` (default: `./regolith_bin_output/`).

## Dependencies
- Python 3
- `numpy`
- `DEME` (PyDEME)

## Key concepts
- **Clump**: a rigid grain shape made from multiple spheres. Clumps approximate irregular regolith grains better than single spheres.
- **Scale (`SCALES`)**: multiplies the *linear size* of a clump. Because volume scales as `scale^3`, mass and inertia scale accordingly.
- **grain_per_scale_probs**: a probability distribution over all (clump shape × size scale) combinations used when spawning grains.

## Configuration highlights
- Material model:
  - `E`, `NU`, `COR`, `MU`, `CRR` define contact stiffness and collision behavior for the terrain material.
- Geometry / grain types:
  - `CLUMP_FILES` defines the clump shapes (sphere layouts).
  - `VOLUMES` and `MASS_MOMENTS_OF_INERTIA` must match the clump files (same order).
- Size distribution:
  - `SCALES` defines the available grain sizes.
  - `PARTICLE_PROBABILITIES` sets relative weighting per size (then converted to number-based probabilities using `scale^3`).
- Performance / stability:
  - `NUM_CLUMPS` controls how many grains are generated total.
  - `BATCH_ADD_CAP` limits how many are added per spawn batch.
  - `STEP_SIZE`, `MAX_VELOCITY`, `COLLISION_BIN_SIZE`, `BOUNDING_BOX_INFLATION` affect stability and runtime.

## How it works (high level)
1. Create a DEM solver and load a terrain material (`LoadMaterial`).
2. For each clump file:
   - Compute `mass = TERRAIN_DENSITY * volume`.
   - Compute `moi` from provided inertia values and density.
   - Load the clump geometry via `DEME.GetDEMEDataFile(...)` and register it as a clump type.
3. Duplicate and scale each clump template for every size in `SCALES`.
4. Build a selection probability distribution over (shape × scale) combinations.
5. Create a bin domain + floor plane.
6. Repeatedly:
   - Sample spawn points using `HCPSampler` (hexagonal close packing) in a thin layer near the top.
   - Randomly assign a template to each point using `grain_per_scale_probs`.
   - Add clumps, give a small initial downward velocity, then settle for `SETTLE_BATCH_TIME`.
   - Save per-frame sphere positions during settling.
7. Run a short final dynamics step, then write `clumps.csv` and `contact_pairs.csv`.

## Notes / tips
- If the simulation is slow: reduce `NUM_CLUMPS`, increase `COLLISION_BIN_SIZE` (carefully), or increase `STEP_SIZE` only if stable.
- If particles “explode”: reduce `E`, reduce `MAX_VELOCITY`, reduce `STEP_SIZE`, or increase `BOUNDING_BOX_INFLATION`.
- If you want a uniform size mix: keep `PARTICLE_PROBABILITIES` equal; the script will still correct for `scale^3` so counts make sense.

## Running
From the directory containing the script:

```bash
python regolith_bin.py

