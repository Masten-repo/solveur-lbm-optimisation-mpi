# TOP-26: Project

**Optimization of an hybrid parallel D2Q9 lattice Boltzmann solver for Kármán vortex street.**

A complete description and instructions for the project are available [here](https://dssgabriel.github.io/CHP203-TOP/project/).

## Requirements

**Simulation:**

- C++ compiler
- CMake 3.25+
- MPI implementation (conforming to MPI 3.0+ standard)
- OpenMP (conforming to OpenMP 4.0+ specification)

**Validation and visualization:**

- Python 3.10+
- [uv](https://github.com/astral-sh/uv)
- Gnuplot
- Compiled `top.display` binary

## Build & Run

**Simulation**

Build:

```bash
# Configure
cmake -B <BUILD_DIR>

# Compile
cmake --build <BUILD_DIR> -t top.lbm-exe
```

Run:

```bash
# Execute with 512 MPI processes
mpirun -np 512 ./<BUILD_DIR>/top.lbm-exe <CONFIG_FILE>
```

**Display helper program**

```bash
# Configure
cmake -B <BUILD_DIR>

# Compile
cmake --build <BUILD_DIR> -t top.display
```

## Configuration file

The simulation is configured using a simple `config.txt` text file in the following format:

```
iterations           = 20000
width                = 800
height               = 160
obstacle_x           = 100.0
obstacle_y           = 80.0
obstacle_r           = 11.0
reynolds             = 100
inflow_max_velocity  = 0.18
output_filename      = results.raw
write_interval       = 100
```

The parameters are:

| Parameter             | Description                                                                                          |
| --------------------- | ---------------------------------------------------------------------------------------------------- |
| `iterations`          | Number of time steps                                                                                 |
| `width`               | Total width of the mesh                                                                              |
| `height`              | Total height of the mesh                                                                             |
| `obstacle_x`          | X-axis position of the obstacle                                                                      |
| `obstacle_y`          | Y-axis position of the obstacle                                                                      |
| `obstacle_r`          | Radius of the obstacle                                                                               |
| `reynolds`            | Ratio of inertial to viscous forces governing the laminar to turbulent transition regime of the flow |
| `inflow_max_velocity` | Maximum inlet flow velocity                                                                          |
| `output_filename`     | Path of the output `.raw` file (no write if undefined)                                               |
| `write_interval`      | Number of time step between writes to the output file                                                |

## Validate & Visualize

An `lbm-viz` tool is provided to help you validate and visualize LBM simulation results as GIFs.

**Local installation**

```bash
uv pip install -e .
```

**Usage**

Compare two files (verify checksums):

```bash
lbm-viz --check ref_results.raw <INPUT>.raw
```

_Run this regularly to validate that your changes don't affect the results of the simulation!_

Generate GIF from `.raw` file:

```bash
lbm-viz --generate-gif <INPUT>.raw <OUTPUT>.gif
```

Extract frames as PNG:

```bash
# Defaults to extracting the last frame
lbm-viz --png <INPUT>.raw <OUTPUT>.png

# Extract specific frame as PNG
lbm-viz --png <INPUT>.raw <OUTPUT>.png --frame 0
```

**Options**

| Option                        | Description                     | Default               |
| ----------------------------- | ------------------------------- | --------------------- |
| `--generate-gif INPUT OUTPUT` | Generate GIF from .raw file     | -                     |
| `--png INPUT OUTPUT`          | Extract frame as PNG            | -                     |
| `--check REFERENCE INPUT`     | Compare INPUT against REFERENCE | -                     |
| `--frame N`                   | Frame index for PNG (0-indexed) | Last frame            |
| `-j, --workers N`             | Number of parallel workers      | Physical cores - 1    |
| `-d, --delay N`               | GIF frame delay (centiseconds)  | 2                     |
| `-s WIDTH HEIGHT`             | Output dimensions for GIF       | Auto (mesh × 1.8)     |
| `--cbr MIN MAX`               | Colorbar range                  | 0.0 - 0.14            |
| `--display-bin-path`          | Path to display binary          | `./build/top.display` |

**Development**

To run directly without installing locally:

```bash
uv run python -m lbm_viz --generate-gif results.raw test.gif
```

**Profile**

Make sure to download hotspot AppImage and put it in

```bash
profiling/bin
```

To profile, for instance record with 8 processus

```bash
./script/build.sh
./profiling/script/run.sh RECORD 8
```

To report,
for instance report perf data from the processus 5

```bash
./profiling/script/run.sh REPORT 5
#or
./profiling/script/run.sh HOTSPOT 5 #for a gui representation

```





# Optimisation d'un solveur Lattice Boltzmann 2D (MPI/OpenMP)

Projet TOP (M1 CHPS, UVSQ – Université Paris-Saclay), réalisé en binôme. Optimisation d'un solveur Lattice Boltzmann (maillage 800×160, 1000 itérations) en C++ avec MPI et OpenMP.

## Méthode

Profilage avec `perf stat` (cycles, défauts de cache L1, dTLB, branch-misses) et MAQAO `lprof`, une optimisation à la fois, métrique de référence : MLUPS (Mega Lattice Updates Per Second).

## Optimisations

1. Suppression d'un verrou de synchronisation artificiel (1 s par itération).
2. Passage de AoS à SoA pour la propagation (38 % du temps CPU, ~11 défauts de cache L1 par cellule).
3. Extraction de l'invariant de boucle u² dans la collision.
4. Suppression des barrières MPI globales (échange de halos point à point).
5. Étude MPI pur vs OpenMP pur (politique First Touch, topologie CCD), contrôle de l'oversubscription, ordonnancement OpenMP static.

## Résultats (8 cœurs, AMD Ryzen 7 7735HS)

| Configuration | MPI | OMP | MLUPS |
|---|---|---|---|
| Baseline (avec verrou) | 1 | 1 | 0,13 |
| Baseline réelle | 1 | 1 | 6,92 |
| Oversubscription | 8 | libre | 7,33 |
| OpenMP pur | 1 | 8 | 17,71 |
| MPI pur (dynamic) | 8 | 1 | 121,01 |
| MPI pur (static) | 8 | 1 | 124,87 |

Gain réel sur le temps de calcul : **×18** (6,92 → 124,87 MLUPS).

## Limites

Les implémentations n'ont pas passé les tests de validation par checksum. Pistes restantes : communications non bloquantes (`MPI_Isend`/`MPI_Irecv`) et topologie cartésienne (`MPI_Cart_create`).

## Rapport

Voir `rapport.pdf`.
