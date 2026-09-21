# CS 598 APE HW1: Optimized Ray Tracer

## Overview

This repository contains the final optimized ray tracer artifact for UIUC CS 598 APE HW1 / Mini-Paper 1. The work used profiling and controlled, incremental performance optimizations while preserving the renderer's command-line interface, input format, workload, and output correctness.

The final version is commit `4a0151f`. The original baseline is commit `19bbc81`.

## Build

### Requirements

- A Linux or compatible Unix environment
- GNU Make
- `g++`
- ImageMagick for reading or writing non-PPM images (`magick` is expected by the program)
- FFmpeg for producing movies from animated workloads
- Git for inspecting the optimization history or creating a baseline worktree

The supplied [`docker/Dockerfile`](docker/Dockerfile) installs the compiler, build tools, ImageMagick, FFmpeg, Git, and profiling-related tools used by the project.

Build the final executable from the repository root:

```bash
make -j
```

The executable is created as `./main.exe`. Core sources under `src/` are compiled with `-O3 -g -Werror` in the final version.

For a clean rebuild:

```bash
make clean
make -j
```

The Docker image can optionally be built from the repository root with:

```bash
docker build -t 598ape-hw1 docker
```

## Running

Run commands from the repository root. The program reports elapsed time as:

```text
Total time to create images=<seconds> seconds
```

### PianoRoom, 500 x 500

```bash
./main.exe -i inputs/pianoroom.ray --ppm -o output/pianoroom.ppm -H 500 -W 500
```

This scene contains a reflecting checkerboard floor, staircase, sphere, rug, walls, and mirror-like surfaces.

### Globe, single frame at 500 x 500

```bash
./main.exe -i inputs/globe.ray --ppm -o output/globe.ppm -H 500 -W 500
```

The repository also includes `inputs/globe.animate` for generating an animated Globe workload.

### Animated sphere mesh, 24 frames at 100 x 100

```bash
./main.exe -i inputs/elephant.ray --ppm -a inputs/elephant.animate --movie -F 24 -W 100 -H 100 -o output/sphere.mp4
```

Despite the input filename, the enabled mesh in `inputs/elephant.ray` is the provided 3,168-triangle sphere mesh loaded from `data/x.txt` and `data/f.txt`. FFmpeg is required when `--movie` is used.

### Elephant mesh, 24 frames at 100 x 100

The original repository instructions refer to `inputs/realelephant.ray`, but that file is not present in the repository. To select the provided 111,748-triangle elephant mesh using the checked-in inputs, edit `inputs/elephant.ray`: comment out the active `data/x.txt 1586 data/f.txt 3168 -1.58 -.43 2.7` line and uncomment `data/elepx.txt 62779 data/elepf.txt 111748 -1.58 -.43 2.7`. Then run:

```bash
./main.exe -i inputs/elephant.ray --ppm -a inputs/elephant.animate --movie -F 24 -W 100 -H 100 -o output/elephant.mp4
```

This workload uses the substantially larger elephant mesh in `data/elepx.txt` and `data/elepf.txt`; no Elephant performance result is claimed here. FFmpeg is required when `--movie` is used.

Other supported options are available with:

```bash
./main.exe --help
```

## Benchmark Methodology

All reported original-to-final comparisons were measured on the same UIUC course VM:

- Ubuntu Linux
- 4 vCPUs
- 16 GB RAM
- 256 GB disk

The same command, resolution, and scene were preserved within each comparison. Warm-up or reference runs were excluded from reported averages. Five formal timed runs were used for the main comparisons. Timing values are taken from the program's `Total time to create images` output.

No CPU model, clock frequency, cache configuration, or unverified hardware detail is assumed here.

## Performance Results

### Original-to-final comparison

| Workload | Original mean | Final mean | Speedup | Runtime reduction |
|---|---:|---:|---:|---:|
| PianoRoom, 500 x 500 | 2.364918 s | 0.754636 s | 3.134x | 68.09% |
| Globe, 500 x 500 | 0.910864 s | 0.414224 s | 2.199x | 54.52% |

### PianoRoom optimization progression

| Version | Mean runtime | Result |
|---|---:|---|
| Original (`19bbc81`) | 2.364918 s | Baseline |
| Optimization #1 (`d3d2a20`) | 1.044108 s | Core compilation with `-O3` |
| Optimization #2 (`40eb34f`) | 0.896477 s | Single-pass nearest-intersection selection |
| Optimization #3 | 0.898468 s | No measurable improvement |
| Optimization #4 (`4a0151f`) | 0.754636 s | Box-specific x/y coordinate calculation |

Optimization #3 was evaluated as an intermediate working-tree state. Its approximately 0.2%-0.3% difference from Optimization #2 was small relative to run-to-run variation and is treated as **no measurable improvement**, not as evidence of a slowdown.

## Optimizations

### Optimization #1: Core compiler optimization

Commit: `d3d2a20`

The core compilation flags in `src/Makefile` were changed from `-O0 -g -Werror` to `-O3 -g -Werror`. The texture sources and final `main.cpp` compilation already used `-O3`.

### Optimization #2: Single-pass nearest-intersection selection

Commit: `40eb34f`

The original `calcColor()` repeatedly allocated a growing array of intersection records, copied previous entries, stored every intersection, sorted the array, and then consumed only the nearest result. The optimized implementation scans the Shape list once and retains only `curTime` and `curShape`, eliminating the repeated allocation, copying, storage, and sorting.

### Optimization #3: Early return for Box plane misses

No separate Git commit was created for this experiment.

`Box::getIntersection()` now returns before coordinate conversion when `Plane::getIntersection()` returns infinity. This avoids calling the coordinate solver for a ray that already missed the supporting plane. The change was benchmarked separately and produced **no measurable performance improvement** on PianoRoom. It is retained in the final Box implementation.

### Optimization #4: Reduced Box coordinate calculation

Commit: `4a0151f`

The general coordinate solver returns x, y, and z values, but the Box boundary and texture calculations consume only x and y. The final Box implementation uses a specialized calculation that retains the required determinant and x/y expressions while omitting the unused z numerator, division, and result. It is used by both `Box::getIntersection()` and `Box::getLightIntersection()`.

Commit `4a0151f` also contains Optimization #3 because that experiment was retained but was not preserved as a separate checkpoint.

## Correctness

Fresh renders from the original baseline and final optimized revision were compared with SHA-256.

| Workload | Original SHA-256 | Final SHA-256 | Result |
|---|---|---|---|
| PianoRoom | `efd0a73f8b2bffde94501ae34e8f29cee99071cb22909455a15d102faac3096a` | `efd0a73f8b2bffde94501ae34e8f29cee99071cb22909455a15d102faac3096a` | Identical |
| Globe | `520fb5682847d2fbb4b219159e186f36ed8d2fe1d3828bf7503eef9a9f8c24b9` | `520fb5682847d2fbb4b219159e186f36ed8d2fe1d3828bf7503eef9a9f8c24b9` | Identical |

For both scenes, the original and final PPM files were byte-for-byte identical.

Example verification command:

```bash
sha256sum output/pianoroom.ppm output/globe.ppm
```

## Reproducing the Baseline Safely

The original baseline is commit `19bbc81`. To avoid changing or overwriting the current checkout, create a separate Git worktree from the repository root:

```bash
git worktree add ../598APE-HW1-baseline 19bbc81
cd ../598APE-HW1-baseline
make clean
make -j
```

Run the same workload command in that worktree and keep its output separate from the final-version output. When the baseline worktree is no longer needed, return to the final repository checkout and remove it with:

```bash
git worktree remove ../598APE-HW1-baseline
```

An independent clone checked out at `19bbc81` is also safe. Avoid resetting the final artifact checkout merely to reproduce the baseline.

## Repository History

| Commit | Description |
|---|---|
| `19bbc81` | Original baseline |
| `d3d2a20` | Optimize core compilation with O3 |
| `40eb34f` | Optimize nearest intersection search |
| `4a0151f` | Optimize Box coordinate calculations |

Optimization #3 does not have a separate commit.

## Notes and Limitations

- Optimization #3 removed unnecessary work but produced no measurable improvement on PianoRoom.
- The full original animated sphere-mesh benchmark was not completed, so no original-to-final sphere-mesh speedup is claimed.
- Comparative speedups should be interpreted only for workloads with both original and final measurements: PianoRoom and Globe.
- Results describe the stated UIUC VM, commands, resolutions, and measured revisions; performance on other machines or workloads may differ.
