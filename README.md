# GutenFree Diet (combolite-demo)

`GutenFree Diet` is an interactive ncurses-based e-reader, ingestion pipeline, and real-world stress test for the `combolite` database engine. It exercises every data model natively supported by the engine against public domain literature from [Project Gutenberg](https://www.gutenberg.org/):

| Feature | combolite Storage Model | Implementation Details |
|---|---|---|
| **E-Book Card Catalog** | Relational Table | ~80,000 titles from `GUTINDEX.ALL` and `pg_catalog.csv` with 8 composite indexes and 7 interactive sort orders. |
| **Passage Storage** | Document Pool | Book texts partitioned into ~2 KB passages with full JSON delta revision histories. |
| **Full-Text Search** | Full-Text Index (BM25) | Inverted index tokenization and BM25 relevance scoring across all downloaded literature. |
| **Semantic Similarity** | Vector Collection & Links | Dense float32 embeddings (384-dimensional) and 1-bit binary vectors with an HNSW-like neighbour graph and `similar_to` graph links. |
| **Telemetry & Metrics** | Time Series | Nanosecond-resolution operation timers, live probe load gauges, and hardware acceleration telemetry. |

> **Note on Data & Models:** No copyrighted books, database archives, or machine learning models are stored in this repository. All datasets and model weights are retrieved locally using the scripts described below.

---

## Prerequisites

- **POSIX OS**: Linux (Debian, Ubuntu, Fedora, Arch) or macOS.
- **Compilers & Build**: C99 compiler (`gcc` or `clang`), C++ compiler (`g++` or `clang++`), CMake >= 3.20.
- **Terminal UI**: `ncurses` development headers with wide-character support:
  - *Debian/Ubuntu*: `sudo apt install libncurses-dev`
  - *Fedora*: `sudo dnf install ncurses-devel`
  - *Arch*: `sudo pacman -S ncurses`
- **Embedding Support**: Python 3 (standard library only) for downloading books and metadata.
- **Hardware Acceleration (Optional)**:
  - NVIDIA CUDA Toolkit >= 11.8 for GPU-accelerated embeddings and vector search.
  - Vulkan SDK 1.3+ with `glslc` for cross-vendor GPU compute.

---

## Detailed Build Guide

### 1. Building via Umbrella Presets (Recommended)

From the repository root, build using the configured presets:

```sh
# Build with CPU fallback (no GPU required)
cmake --preset gutenfree-cpu
cmake --build --preset gutenfree-cpu -j

# Build with NVIDIA CUDA acceleration (requires nvcc and CUDA toolkit)
cmake --preset gutenfree-cuda
cmake --build --preset gutenfree-cuda -j

# Build with Vulkan compute acceleration
cmake --preset gutenfree-vulkan
cmake --build --preset gutenfree-vulkan -j

# Build with both CUDA and Vulkan backends enabled
cmake --preset gutenfree-both
cmake --build --preset gutenfree-both -j
```

The resulting executable is generated in the corresponding build directory (e.g., `build-gutenfree-cpu/combolite-demo/gutenfree`).

### 2. Standalone Build

To build directly from within `combolite-demo/`:

```sh
cd combolite-demo
cmake -B build -DCMAKE_BUILD_TYPE=RelWithDebInfo
cmake --build build -j
```

---

## Getting Started: Ingestion and E-Reader

All operations run from within `combolite-demo/` or use the relative paths to `data/`. All fetched books, metadata, and generated database archives are stored in `data/` (which is git-ignored).

### 1. Fetch Metadata and Embedding Model

Download scripts retrieve files strictly from Project Gutenberg mirrors (default: `pglaf.org`):

```sh
cd combolite-demo

# 1. Fetch the master catalog index (GUTINDEX.ALL, ~9 MB)
python3 gf_fetch.py --gutindex

# 2. Fetch extended catalog metadata (genres, dates, LoCC classes, ~21 MB)
python3 gf_fetch.py --catalog

# 3. Download the verified embedding model (bge-small-en-v1.5, ~37 MB, SHA-256 checked)
python3 gf_model.py

# 4. Fetch a sample pool of books (e.g. 25 random English books)
python3 gf_fetch.py --count 25
```

### 2. Ingest and Embed

Run the compiled `gutenfree` binary to populate the database:

```sh
# Ingest catalog metadata into the relational table (~30 seconds)
./gutenfree ingest-index

# Parse downloaded books into passages and build the BM25 full-text index
./gutenfree ingest-books

# Compute vector embeddings, construct the neighbour graph, and link similar books
./gutenfree embed
```

The database archive is saved to `data/gutenfree.cdbarch` with companion runtime files in `data/gutenfree.cdbwork/`. To specify a different database path, set `GUTENFREE_DB=/custom/path`.

### 3. Launch the Interactive E-Reader

Launch the ncurses interface:

```sh
./gutenfree
```

#### Keybindings Navigation

| Key | View | Description |
|---|---|---|
| `c` | **Card Catalog** | Browse ~80,000 books. `/` filter, `o` change sort, `Space` queue download, `Enter` details. |
| `r` | **Reading Shelf** | View and read stored passages of downloaded books. |
| `s` | **Full-Text Search** | Query book text with BM25 scoring and snippet highlights. |
| `v` | **Semantic Search** | Compare exact cosine search, approximate neighbour-graph search, and 1-bit Hamming search. |
| `d` / `F2` | **Diagnostics** | Live telemetry, latency percentiles (p50, p95, p99), engine buffer gauges, and accelerator utilization. |
| `e` | **Embed Books** | Trigger batch vector embedding for freshly ingested books. |
| `q` | **Quit** | Cleanly close database and flush coordinator slots. |

---

## Parallel Stress Testing: The Ingestion Pipeline

The demo includes a stress pipeline designed to validate engine concurrency, multi-process coordinator locking, and crash safety under sustained write load:

```sh
# Run parallel ingestion pipeline with 4 independent writer processes
GUTENFREE_WORKERS=4 ./gutenfree pipeline 100
```

### How the Pipeline Works:
- **Concurrent Ingestion**: Concurrently downloads up to 100 unseen books and distributes them across `GUTENFREE_WORKERS` parallel worker processes.
- **Independent Database Handles**: Each worker process opens an independent `cdb_db` instance with its own coordinator slot file.
- **Hardware-Accelerated Embedding**: Workers compute 384-dimensional embeddings on the GPU and commit vector records concurrently.
- **Integrity Validation**: One out of every ten books is held out to generate unseen semantic search test queries (`gf_queries.c`), verifying exact vs. approximate recall in real time.

---

## Benchmarking & Diagnostics Commands

```sh
# Apply query filters directly from the command line
./gutenfree filter genre="science fiction" author=verne

# Run 15 seconds of synthetic probe load (reads, point scans, vector searches)
./gutenfree probe 15

# Display aggregated latency metrics across all database operations
./gutenfree metrics

# Benchmark neighbour graph construction time on a scratch database copy
./gutenfree bench-graph data/test_copy
```

---

## Features as Built

- **Multi-Process Concurrency**: Full compatibility with the liveness redesign (steps 1–5, 7); writers hold OS slot file locks and automatically adopt pending journals if a sibling process terminates abruptly.
- **Hardware Accelerator Support**: Automatic runtime dispatch to NVIDIA CUDA or Vulkan GPU backends for dense vector similarity scoring and neighbour graph refreshes.
- **Real-Time Telemetry**: Microsecond-resolution timing instrumentation recording call count, min, p50, p95, p99, and max durations for every database engine call.
- **Robust Network Fetching**: Backoff logic handling HTTP 429/503 responses and mirror failovers during automated book retrieval.

---

## Known Issues

1. **Full-Text Memory Footprint at Large `doc_max`**: The token parser enforces zero-cut token guarantees, requiring up to 260 MB of token buffer space when `doc_max` is configured to 8 MB.
2. **Idle Collection Log Lag**: If a vector collection remains closed during database reconciliation, its change-log trimming is deferred until its next open.
3. **GPU Power States Under Sparse Vulkan Workloads**: Short, sporadic Vulkan search bursts on NVIDIA hardware can cause driver power downclocking (P8 state), resulting in latency spikes compared to persistent CUDA contexts.

---

## ToDos

- Integrate upcoming CMBQL query compiler when available in core engine.
- Implement automatic worker count sizing based on host CPU core availability and VRAM limits.
- Add support for dynamic model quantization (e.g. 4-bit / 8-bit weights) in the embedding inference runner.

---

## Provisioned but Not Yet Implemented

1. **High-Level CMBQL Query Integration**: The demo currently issues operations directly through internal C99 engine APIs (`cdb_table_*`, `cdb_doc_*`, `cdb_vec_*`). Pipelined CMBQL query string execution is provisioned for future integration once the upstream compiler is stabilized.
2. **Automated Peer-to-Peer Catalog Synchronization**: P2P replication gossip protocols (spec 19 draft) are not yet integrated into the demo application; catalog synchronization currently operates via batch metadata ingestion.

---

## License & Third-Party Attributions

- Demo code: MIT License.
- `llama.cpp`: MIT License (vendored in `demo/third-party/llama.cpp`, pinned to b11701).
- `bge-small-en-v1.5`: MIT License (BAAI / CompendiumLabs).
- `ncurses`: X11 / MIT-compatible license.
- E-book texts and catalog datasets: Public domain from Project Gutenberg. See [`ATTRIBUTIONS.md`](../ATTRIBUTIONS.md) for full licensing records.
