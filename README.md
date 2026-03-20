# nalimov-rs

Zero-dependency, pure Rust implementation of [Nalimov endgame tablebase](https://en.wikipedia.org/wiki/Endgame_tablebase#Nalimov) probing.
Reads `.nbw.emd` / `.nbb.emd` compressed files and returns exact distance-to-mate (DTM) values.

- **3–6 piece** endgames with en passant support
- **Thread-safe** — `Mutex` + O(1) LRU block cache (256 MB)
- **Faster** than the original C++ implementation on cache-hot probes
- **Zero external crate dependencies** — pure `std` only
- **C API** — build as a static library, link from C/C++

## Downloading tablebase files

Nalimov tablebase files (`.nbw.emd` / `.nbb.emd`) can be downloaded from:

- **[tablebase.sesse.net](https://tablebase.sesse.net)** — Steffen Kjær Jacobsen's mirror
- **[Hugging Face](https://huggingface.co/datasets/jromanghf/nalimov-tablebases)** — `jromanghf/nalimov-tablebases` dataset

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

let prober = NalimovProber::new(&["/path/to/nalimov/3-4-5"]).unwrap();

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
