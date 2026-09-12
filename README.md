# nalimov-rs

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Rust](https://img.shields.io/badge/Rust-1.85+-orange.svg)](https://www.rust-lang.org)

Zero-dependency, pure Rust implementation of [Nalimov endgame tablebase](https://en.wikipedia.org/wiki/Endgame_tablebase#Nalimov) probing.
Reads `.nbw.emd` / `.nbb.emd` compressed files and returns exact distance-to-mate (DTM) values.

- **3–6 piece** endgames with en passant support
- **Thread-safe** — `Mutex` + configurable O(1) LRU block cache
- **Faster** than the original C++ implementation on cache-hot probes
- **Zero external crate dependencies** — pure `std` only
- **C API** — build as a static library, link from C/C++

## Why Nalimov in 2026?

[Syzygy tablebases](https://github.com/syzygy1/tb) have become the de facto standard in computer chess, and
rightly so — Ronald de Man's work is a piece of art: compact, elegant, and
perfectly suited to the needs of modern engines with their DTZ metric and
50-move-rule awareness. There is no question that Syzygy tables are a
remarkable achievement.

Yet Nalimov tables have a quality of their own that deserves not to be
forgotten. They store **distance-to-mate (DTM)** — the one metric that answers
the most natural question in chess: *"how many moves until checkmate?"* DTM
gives you the shortest path to mate, unconditionally. No conversion, no
ambiguity — just a ply count.

[Gaviota tablebases](https://github.com/michiguel/Gaviota-Tablebases) (by Miguel Ballicora) also store DTM and come with a
permissive license, but they only cover up to **5 pieces**. Nalimov tables go
up to **6 pieces**, which matters for many practical endgames.

DTM matters in practice:

- **Endgame study and analysis** — DTM is the gold standard for verifying
  composed studies, analyzing adjudicated games, and understanding the true
  difficulty of an endgame.
- **Engine play** — with DTM the engine can always prefer the fastest mate,
  producing cleaner, more convincing endgame play without the heuristic
  workarounds that DTZ sometimes requires.
- **Simplicity of integration** — a single `probe()` call returns a signed ply
  count. Straightforward to wire into any search.

The original C++ probing code served the community well for over two decades.
This crate aims to make Nalimov probing more accessible: a clean, memory-safe,
single-file Rust implementation with zero dependencies, thread safety built in,
and an **MIT license** that allows unrestricted use in any project — open source
or proprietary.

Nalimov tables are a piece of chess programming history. Hopefully this crate
will help keep them alive.

## Downloading tablebase files

Nalimov tablebase files (`.nbw.emd` / `.nbb.emd`) can be downloaded from:

- **[tablebase.sesse.net](https://tablebase.sesse.net)** — Steffen Kjær Jacobsen's mirror
- **[Hugging Face](https://huggingface.co/datasets/jromanghf/nalimov-tablebases)** — `jromanghf/nalimov-tablebases` dataset

To download from Hugging Face, install the [`hf` CLI](https://huggingface.co/docs/huggingface_hub/guides/cli) and run:

```sh
# Install the CLI (pick one)
curl -LsSf https://hf.co/cli/install.sh | bash   # standalone installer
pip install -U huggingface_hub                     # or via pip

# Download all tablebase files
hf download --repo-type dataset jromanghf/nalimov-tablebases --local-dir nalimov-tb
```

## Usage: single-file drop-in

Copy `src/nalimov.rs` into your project's `src/` directory and add to your `lib.rs` or `main.rs`:

```rust
pub mod nalimov;
```

No Cargo dependencies needed.

## Usage: Rust crate

Add to your `Cargo.toml`:

```toml
[dependencies]
nalimov = { git = "https://github.com/jromang/nalimov-rs" }
```

```rust
use nalimov::{NalimovProber, NalimovResult};

let prober = NalimovProber::new(&["/path/to/nalimov/3-4-5"], 256).unwrap();

// KQvK: White Ke1(4) Qd1(3), Black Kh8(63) — White to move
// Square mapping: a1=0, b1=1, …, h1=7, a2=8, …, h8=63 (LERF)
let result = prober.probe(
    (1u64 << 4) | (1u64 << 3),    // white occupancy
    1u64 << 63,                     // black occupancy
    (1u64 << 4) | (1u64 << 63),    // kings (both colors)
    1u64 << 3,                      // queens
    0, 0, 0, 0,                     // rooks, bishops, knights, pawns
    true,                            // white to move
    64,                              // no en passant (64 = none)
);

match result {
    Ok(NalimovResult::Win { plies })  => println!("Mate in {plies} plies"),
    Ok(NalimovResult::Loss { plies }) => println!("Mated in {plies} plies"),
    Ok(NalimovResult::Draw)           => println!("Draw"),
    Err(e)                            => println!("Probe failed: {e:?}"),
}
```

## Usage: C/C++

Build the static library:

```sh
cargo build --release
# Produces: target/release/libnalimov.a
```

Declare the API in your C code (no header file shipped):

```c
#include <stdint.h>

// Initialize tablebases from a directory path.
// Returns 1 on success, 0 on failure.
int rs_nalimov_init(const char *path);

// Probe a position using bitboards (LERF: a1=0, h8=63).
// Each piece-type bitboard contains squares for BOTH colors.
// wtm: 1 = white to move, 0 = black to move.
// ep_square: en passant target (0–63), or 64 if none.
// out_score: >0 = side to move wins in N plies,
//            <0 = side to move loses in N plies, 0 = draw.
// Returns 1 on success, 0 on failure.
int rs_nalimov_probe(
    uint64_t white, uint64_t black,
    uint64_t kings, uint64_t queens, uint64_t rooks,
    uint64_t bishops, uint64_t knights, uint64_t pawns,
    int wtm, uint8_t ep_square, int *out_score);
```

Link:

```sh
gcc -O2 -o myengine myengine.c -L target/release -lnalimov -lpthread -ldl -lm
```

Example:

```c
#include <stdio.h>
#include <stdint.h>

extern int rs_nalimov_init(const char *path);
extern int rs_nalimov_probe(uint64_t, uint64_t, uint64_t, uint64_t,
    uint64_t, uint64_t, uint64_t, uint64_t, int, uint8_t, int *);

int main() {
    rs_nalimov_init("/path/to/nalimov/3-4-5");

    // KRvK: White Ke1(4) Rh1(7), Black Ka8(56)
    uint64_t white = (1ULL << 4) | (1ULL << 7);
    uint64_t black = (1ULL << 56);
    uint64_t kings = (1ULL << 4) | (1ULL << 56);
    uint64_t rooks = (1ULL << 7);
    int score;

    if (rs_nalimov_probe(white, black, kings, 0, rooks, 0, 0, 0, 1, 64, &score))
        printf("Score: %d\n", score);  // positive = White wins
    return 0;
}
```

## Probing from a chess engine

### Bitboard mapping

The prober uses [LERF](https://www.chessprogramming.org/Square_Mapping_Considerations#Little-Endian_Rank-File_Mapping)
(Little-Endian Rank-File) square mapping: `a1 = 0, b1 = 1, …, h1 = 7,
a2 = 8, …, h8 = 63`. A square is represented by setting its corresponding bit
in a `u64`:

```
 8 | 56 57 58 59 60 61 62 63
 7 | 48 49 50 51 52 53 54 55
 6 | 40 41 42 43 44 45 46 47
 5 | 32 33 34 35 36 37 38 39
 4 | 24 25 26 27 28 29 30 31
 3 | 16 17 18 19 20 21 22 23
 2 |  8  9 10 11 12 13 14 15
 1 |  0  1  2  3  4  5  6  7
     a  b  c  d  e  f  g  h
```

If your engine already uses LERF bitboards (most modern engines do), you can
pass them directly. Otherwise, convert your square indices before calling
`probe()`.

### Building the bitboard arguments

`probe()` takes two **occupancy** bitboards and six **piece-type** bitboards:

| Parameter | What to set |
|-----------|-------------|
| `white` | One bit per square occupied by a white piece (including the king) |
| `black` | One bit per square occupied by a black piece (including the king) |
| `kings` | White king square **OR** black king square |
| `queens` | All queen squares, both colors |
| `rooks` | All rook squares, both colors |
| `bishops` | All bishop squares, both colors |
| `knights` | All knight squares, both colors |
| `pawns` | All pawn squares, both colors |

Piece-type bitboards contain squares for **both colors combined**. The prober
internally separates them with `queens & white`, `pawns & black`, etc.

Example — KRB vs KN, White Ke1 Ra1 Bc1, Black Ke8 Nb8:

```rust
let e1 = 4;  let a1 = 0;  let c1 = 2;
let e8 = 60; let b8 = 57;

let white   = (1u64 << e1) | (1u64 << a1) | (1u64 << c1);
let black   = (1u64 << e8) | (1u64 << b8);
let kings   = (1u64 << e1) | (1u64 << e8);
let rooks   = 1u64 << a1;
let bishops = 1u64 << c1;
let knights = 1u64 << b8;

let result = prober.probe(
    white, black, kings, 0, rooks, bishops, knights, 0,
    true,  // white to move
    64,    // no en passant
);
```

### En passant

Pass the **target square** (the square the capturing pawn would land on), not
the square of the pawn being captured. For example, if Black just played d7-d5,
the en passant square is `d6 = 43`. Pass `64` when there is no en passant.

### When to probe

Nalimov tables cover positions with **3 to 6 pieces** (including kings). A
typical integration in the search looks like:

```rust
fn search(&self, pos: &Position, alpha: i32, beta: i32, depth: i32) -> i32 {
    // Probe the tablebase when few enough pieces remain
    if pos.piece_count() <= 6 {
        match self.prober.probe(
            pos.white(), pos.black(),
            pos.kings(), pos.queens(), pos.rooks(),
            pos.bishops(), pos.knights(), pos.pawns(),
            pos.white_to_move(),
            pos.ep_square(),       // 0–63, or 64 if none
        ) {
            Ok(NalimovResult::Win { plies }) => {
                // Side to move wins: return a high score adjusted by DTM
                return MATE_SCORE - plies as i32;
            }
            Ok(NalimovResult::Loss { plies }) => {
                // Side to move loses: return a low score adjusted by DTM
                return -MATE_SCORE + plies as i32;
            }
            Ok(NalimovResult::Draw) => {
                return DRAW_SCORE;
            }
            Err(_) => {
                // Table not available — continue normal search
            }
        }
    }
    // ... normal alpha-beta search
}
```

**Tips:**

- Probe at the **root** and at **interior nodes** where `piece_count <= 6`.
  Root probes give exact scores; interior probes give perfect pruning.
- `plies` is the **distance to mate** from the probed position, assuming
  optimal play from both sides. Use it to prefer shorter mates.
- `PositionNotFound` usually means the tablebase file for that material
  combination is not installed — the engine should fall back to normal search.
- The prober is **thread-safe**: a single `NalimovProber` instance can be shared
  across search threads (wrap in `Arc`). The internal LRU cache uses a `Mutex`.

### Performance considerations

- The first probe for a given material will read the file header from disk.
  Subsequent probes for the same material hit the in-memory block cache.
- The LRU cache size is selected at construction in MiB. A 256 MiB cache holds
  32 768 decompressed blocks of 8 KiB each.
  Cache-hot probes are fast — no I/O or decompression.
- For best results, place tablebase files on an SSD and probe only when
  `piece_count` is within the range of your installed tables.

## Advantages over the C++ Nalimov prober

| | nalimov-rs | C++ reference |
|---|---|---|
| Language | Pure Rust, memory-safe | C++ with raw pointers |
| Dependencies | Zero | — |
| Thread safety | Built-in (`Mutex` + LRU) | Requires external locking |
| Performance | Faster on cache-hot probes | Baseline |
| 6-piece support | Yes (split extents, lazy I/O) | Yes |
| En passant | Yes | Yes |
| Embeddability | Single file, no build system needed | Multiple `.cpp` / `.h` files |
| C API | `rs_nalimov_init` / `rs_nalimov_probe` | Native |

## Acknowledgements

The tablebase format and probing algorithms were designed by **Eugene Nalimov** (1998–2001).
The LZ77+Huffman compression (DATACOMP v1.0) is by **Andrew Kadatch** (1991–1998).
This crate is a clean-room Rust translation of the C++ reference code.

## License

MIT — see [LICENSE](LICENSE).
