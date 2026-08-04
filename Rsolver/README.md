# Rubik's Cube Solver

A C++ Rubik's Cube solver that scans a physical, scrambled cube using a webcam and computes a near-optimal sequence of moves to solve it. The project explores multiple cube state representations, multiple search algorithms, and a pattern-database heuristic to make optimal search computationally feasible.

## Overview

Solving a Rubik's Cube optimally is a hard search problem — the cube has roughly 4.3 × 10¹⁹ possible states, far too many to search exhaustively. This project implements a progression of search algorithms, from brute-force to heuristic-guided, to make solving tractable:

1. **Scan** a physical cube's face colors via webcam (OpenCV)
2. **Represent** the resulting state internally
3. **Search** for a solution using IDA\*, guided by a precomputed pattern-database heuristic
4. **Output** the move sequence needed to solve it

## Features

- **Three cube state representations** (for performance comparison):
  - `RubiksCube3dArray` — straightforward 3D array of stickers
  - `RubiksCube1dArray` — flattened 1D array for better cache locality
  - `RubiksCubeBitboard` — bit-packed representation for fast comparisons/hashing (used in the final solving pipeline)
- **Four search algorithms**:
  - Depth-First Search (DFS)
  - Breadth-First Search (BFS)
  - Iterative Deepening DFS (IDDFS)
  - Iterative Deepening A* (IDA*) — the algorithm actually used to solve real scrambles
- **Corner pattern database** — a precomputed heuristic table (built via BFS from the solved state) that gives IDA\* an admissible lower bound on moves remaining, dramatically pruning the search
- **Webcam cube scanner** — captures the 6 faces of a physical cube via OpenCV and reconstructs its state automatically, instead of manual color entry

## Project Structure

```
Model/               Cube state representations (3D array, 1D array, bitboard)
Solver/               Search algorithms (DFS, BFS, IDDFS, IDA*)
PatternDatabases/     Pattern database generation and lookup (corner heuristic)
Databases/            Generated pattern database files (e.g. cornerDepth5V1.txt)
Scanner/               Webcam-based cube scanner (OpenCV)
main.cpp              Entry point wiring scanning → solving → output
CMakeLists.txt        Build configuration
```

## How It Works

1. `CubeScanner` opens a webcam feed and detects the sticker colors across all 6 faces of a physical cube.
2. The detected colors populate a `RubiksCubeBitboard` representing the scrambled cube.
3. `IDAstarSolver` searches for a solution, using the precomputed corner pattern database as its heuristic (`f(n) = g(n) + h(n)`), pruning branches that cannot lead to a shorter solution than one already found.
4. The resulting move sequence is printed in standard cube notation (`U`, `U'`, `U2`, `D`, `D'`, `D2`, `L`, `L'`, `L2`, `R`, `R'`, `R2`, `F`, `F'`, `F2`, `B`, `B'`, `B2`).

### Why a pattern database?

Solving the full cube optimally with brute-force search is infeasible — BFS runs out of memory past ~6 moves, and DFS/IDDFS have no way to know which branches are promising. The corner pattern database relaxes the problem: it precomputes, for every possible arrangement of just the 8 corner pieces (ignoring edges), the exact minimum number of moves to solve them. Since solving the full cube also solves its corners, this precomputed value is a guaranteed lower bound (admissible heuristic) on the true solution length — letting IDA\* skip large portions of the search space without sacrificing optimality.

- Corner state space: 8! × 3⁷ = 88,179,840 possible configurations
- Built once via a breadth-first search starting from the solved state, storing "distance from solved" for every reachable corner configuration
- Stored as 4-bit values (max distance ≈ 11 moves) to keep the table around ~42 MB instead of ~84 MB

## Prerequisites

- CMake ≥ 3.20
- A C++14-compatible compiler
- [OpenCV](https://opencv.org/) (required for the webcam scanner)

## Building

```bash
git clone https://github.com/pranav-harresh-v/rubiks-cube-solver.git
cd rubiks-cube-solver
mkdir build && cd build
cmake ..
make
```

## Running

```bash
./rubiks_cube_solver
```

On launch, the program will:

1. Open the webcam and prompt you to scan each face of the physical cube
2. Print the detected scrambled state
3. Run IDA\* (using the corner pattern database) to compute a solution
4. Print the sequence of moves needed to solve the cube

> Note: the corner pattern database file must exist before solving (see below). Generating it is a one-time step; solving is fast afterward.

### Generating the pattern database

If `Databases/cornerDepth5V1.txt` does not already exist, it must be generated once:

```cpp
CornerDBMaker dbMaker(fileName, 0x99);
dbMaker.bfsAndStore();
```

This performs an exhaustive BFS from the solved state over all 88,179,840 corner configurations and writes the resulting distance table to disk. This only needs to be run once — subsequent solves simply load the file.

## Algorithms at a Glance

| Algorithm | Optimal? | Memory                  | Notes                                                                                                              |
| --------- | -------- | ----------------------- | ------------------------------------------------------------------------------------------------------------------ |
| DFS       | No       | Low (O(depth))          | Fast but may return a longer-than-necessary solution                                                               |
| BFS       | Yes      | Very high (O(18^depth)) | Guaranteed shortest solution, but infeasible beyond ~6 moves                                                       |
| IDDFS     | Yes      | Low                     | Repeats DFS at increasing depth limits; optimal but still brute-force                                              |
| IDA\*     | Yes      | Low                     | Same as IDDFS, but guided by the pattern-database heuristic — the only algorithm here practical for real scrambles |

## Acknowledgements

- Pattern database technique based on the approach described in Richard E. Korf's _"Finding Optimal Solutions to Rubik's Cube Using Pattern Databases"_

## License

Specify a license here (e.g. MIT) if you intend this project to be reused by others.
