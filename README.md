# gxgr — The Go-Rust Framework
### Complete Architecture & Design Specification v0.2

> **gxgr** is a hybrid systems framework that harnesses Go's concurrency strengths  
> inside a Rust-owned, memory-safe supervision model.  
> **2 unsafe blocks total. Static binary via musl+zig. Systemd-grade reliability.**

---

## 1. Why gxgr Exists

Modern systems software in pure Rust hits the same wall every time: async concurrency requires implementing Wakers, Pin, task vtables, intrusive linked lists — all requiring `unsafe`. Tokio alone has **~100-200 unsafe blocks** scattered across its source. The Tokio team even added `#![allow(unsafe_op_in_unsafe_fn)]` just to manage the noise.

gxgr's thesis:

> Don't make Rust do what Go already solved in 2009.  
> Don't let Go own what Rust should own.  
> Define a clean enforced contract between them.  
> 2 unsafe blocks. Both in one file. Both auditable in 5 minutes.

---

## 2. Naming & Module Structure

```
gxgr          →  the framework name (Go-Rust)
gr            →  the umbrella pathway concept
crates.io:    gxgr          (search "gx" finds it)
go mod:       gx-modname    (search "gx-" finds all modules)
```

### Module files

```
gxo.rs    →  📋 Object registry (pure static, compile-time only)
              templates, gxN registration, rules
              zero runtime, zero allocation, zero process
              compiles as read-only const data into binary

gxr.rs    →  🎛️ Runner/enforcer (ephemeral coordinator)
              gxr start / stop / restart / status
              reads gxo, coordinates gxN
              boots briefly, steps back — not a daemon

gx1.rs    →  ✋ Supervisor instance 1 (independent hand)
gx2.rs    →  ✋ Supervisor instance 2 (independent hand)
gxN.rs    →  ✋ Supervisor instance N (as many as needed)
```

### Why no singleton

```
old gaps0 singleton:
→ one crash = everything down
→ one bottleneck for all allocation
→ monolithic, hard to scale

gxgr model:
→ gxo = constitution (can't crash, it's just data)
→ gxr = ephemeral coordinator (starts, steps back)
→ gxN = independent hands (each owns its own world)
→ gx1 crashes? gx2 and gx3 keep running
→ gxr restarts gx1 from gxo definition
```

---

## 3. ga — Allocation Template System

Every Go worker hatch is given a **ga template** — a predefined contract
defining exactly how much memory and how many goroutines it may use.
**Workers never allocate. They request a template from gxo.**

### 3.1 Naming Convention

```
ga + unit prefix + size = the template name
the name IS the spec — zero ambiguity

gab   →  bytes
gak   →  kilobytes
gam   →  megabytes
gag   →  gigabytes / nested Rust call context
```

### 3.2 Complete Template Range

#### Byte tier (sub-KB)
| Template | Arena | Goroutines | Channels | Use Case |
|----------|-------|------------|----------|----------|
| `gab1` | 1B | 1 | 1 | single flag / WATCHDOG=1 ping |
| `gab4` | 4B | 1 | 1 | u32 signal with context |
| `gab64` | 64B | 1 | 1 | small status probe |
| `gab128` | 128B | 1 | 1 | healthcheck response |
| `gab256` | 256B | 1 | 1 | event with metadata |
| `gab512` | 512B | 2 | 1 | small message payload |

#### Kilobyte tier (standard workers)
| Template | Arena | Goroutines | Channels | Use Case |
|----------|-------|------------|----------|----------|
| `gak4` | 4KB | 2 | 1 | minimal worker |
| `gak8` | 8KB | 4 | 2 | lightweight worker |
| `gak16` | 16KB | 8 | 4 | standard worker (sweet spot) |
| `gak32` | 32KB | 16 | 8 | heavy worker |
| `gak64` | 64KB | 32 | 16 | data pipeline |
| `gak128` | 128KB | 64 | 32 | large workload |

#### Megabyte tier (think first)
| Template | Arena | Goroutines | Channels | Use Case |
|----------|-------|------------|----------|----------|
| `gam1` | 1MB | 32 | 16 | large file chunk processing |
| `gam4` | 4MB | 64 | 32 | heavy streaming I/O |
| `gam10` | 10MB | 64 | 32 | ⚠️ think twice — chunk instead |
| `gam128` | 128MB | 128 | 64 | 🚨 reconsider architecture |

#### Gigabyte tier (special)
| Template | Arena | Use Case |
|----------|-------|----------|
| `gag1` | nested Rust | Go hatch calls back into Rust |
| `gag4` | nested Rust (4 slots) | Go hatch with 4 parallel Rust calls |
| `gag16` | nested Rust (16 slots) | heavy nested computation |

> `gag` is NOT a memory size — it's a **nested Rust execution context**  
> called FROM inside a Go hatch, back into Rust.  
> See section 6 for details.

### 3.3 Rule: sub-KB = leaf only

```
gab tier:
→ can_spawn = false  (leaf nodes, never parents)
→ do one thing, signal done, exit
→ too tiny to supervise children

gak and above:
→ can_spawn = true   (can be gp parents)
→ standard worker hierarchy
```

### 3.4 Rule: large data stays in Rust

```
Rust native (std::fs, memmap2, serde):
→ file I/O          ✅
→ large reads       ✅
→ CPU computation   ✅
→ cryptography      ✅
→ parsing           ✅
→ data structures   ✅

Go hatch (via ga arena):
→ concurrent I/O    ✅
→ network handling  ✅
→ event loops       ✅
→ goroutine fan-out ✅
→ channel pipelines ✅

gam/gag tier:
→ edge cases only
→ if you need MB arenas, chunk into gak workers first
→ Rust native memory is right there 👉
```

---

## 4. Worker Address Convention

Every Go worker hatch has a three-part address:

```
gx{supervisor} : ga{template} gp{parent} gs{step}

gx  =  which gxN supervisor owns this worker
ga  =  allocation template (memory + cycle budget)
gp  =  group parent (who spawned this worker)
gs  =  group step (sequence within parent group)
```

### Examples

```
gx1:gak16gp1gs1   →  supervisor 1, 16KB arena, parent 1, step 1
gx1:gak16gp1gs2   →  supervisor 1, 16KB arena, parent 1, step 2
gx1:gak8gp2gs1    →  supervisor 1, 8KB arena,  child of gp1, step 1
gx2:gak32gp1gs1   →  supervisor 2, 32KB arena, parent 1, step 1
```

### Hierarchy example

```
gxr
├── gx1 (network supervisor)
│     ├── gx1:gak16gp1gs1   (listener)
│     │     ├── gx1:gak8gp2gs1  (conn handler)
│     │     └── gx1:gak8gp2gs2  (conn handler)
│     └── gx1:gak16gp1gs2   (event processor)
│
└── gx2 (I/O supervisor)
      ├── gx2:gak32gp1gs1   (read pipeline)
      │     ├── gx2:gak16gp2gs1  (read buffer)
      │     └── gx2:gak16gp2gs2  (write buffer)
      └── gx2:gab128gp1gs2  (healthcheck probe)
```

---

## 5. Memory Model — Pinned Arena Protocol

> **Go never owns memory. gxo/gxN owns everything.**

```
1. gxN allocates pinned arena from its pool (Rust)
2. gxN hands bounded slice pointer + size to Go hatch
3. Go worker operates ONLY within that slice
4. Go's GC cannot move pinned memory → no unsafe needed
5. Go worker signals "done" to gxN
6. gxN reclaims arena back to pool
7. Rust ownership fully restored
```

### Arena lifecycle

```
gxN pool:  [ FREE ][ FREE ][ FREE ][ FREE ]

spawn gak16:
gxN pool:  [ FREE ][ IN USE → gp1gs1 ][ FREE ][ FREE ]

gp1gs1 signals done:
gxN pool:  [ FREE ][ FREE ][ FREE ][ FREE ]
```

### Reaping rules

| Event | gxN Action |
|-------|-----------|
| Worker signals Done | mark Done, reclaim arena |
| Timeout exceeded | force-kill goroutine, reclaim arena |
| Cycle budget exceeded | throttle → warn → reap |
| Go panic inside hatch | catch at boundary, reclaim arena, optionally respawn |
| Parent group reaped | walk gp tree, reap all gs children |

---

## 6. Nested Rust Calls — gag tier

`gag` templates enable a Go hatch to **call back into Rust** for
CPU-heavy work without blocking the goroutine scheduler.

```
normal flow (gak):
gxr → gx1 → Go hatch → work → done

nested flow (gag):
gxr → gx1 → Go hatch → gag::call_rust() → Rust computes → returns to Go → done
```

### When to use gag

```
Go hatch doing concurrent network work (gak16)
needs heavy CPU computation mid-flight:
→ crypto, parsing, compression
→ don't block the goroutine
→ don't do CPU work in Go (Go loses here)
→ gag::call_rust() → fast Rust computation
→ result returned to Go hatch
→ Go continues concurrent work
```

### Nesting depth limit

```
gxr (Rust)
 └── gak (Go hatch)        level 1
       └── gag (Rust call) level 2 ← MAX

beyond level 2:
→ rejected by gxr at runtime
→ "rethink your architecture" 😄
```

---

## 7. gxo.rs — The Registry

Pure static Rust. Zero runtime. Zero process. Compiles as read-only const data.

```rust
// gxo.rs — complete ga template registry

// ── byte tier ─────────────────────────────
pub const GAB1:   GaTemplate = GaTemplate { name:"gab1",   arena:1,       goroutines:1,  channels:1,  can_spawn:false, priority:255 };
pub const GAB4:   GaTemplate = GaTemplate { name:"gab4",   arena:4,       goroutines:1,  channels:1,  can_spawn:false, priority:254 };
pub const GAB64:  GaTemplate = GaTemplate { name:"gab64",  arena:64,      goroutines:1,  channels:1,  can_spawn:false, priority:253 };
pub const GAB128: GaTemplate = GaTemplate { name:"gab128", arena:128,     goroutines:1,  channels:1,  can_spawn:false, priority:252 };
pub const GAB256: GaTemplate = GaTemplate { name:"gab256", arena:256,     goroutines:1,  channels:1,  can_spawn:false, priority:251 };
pub const GAB512: GaTemplate = GaTemplate { name:"gab512", arena:512,     goroutines:2,  channels:1,  can_spawn:false, priority:250 };

// ── KB tier ───────────────────────────────
pub const GAK4:   GaTemplate = GaTemplate { name:"gak4",   arena:4_096,   goroutines:2,  channels:1,  can_spawn:false, priority:200 };
pub const GAK8:   GaTemplate = GaTemplate { name:"gak8",   arena:8_192,   goroutines:4,  channels:2,  can_spawn:true,  priority:180 };
pub const GAK16:  GaTemplate = GaTemplate { name:"gak16",  arena:16_384,  goroutines:8,  channels:4,  can_spawn:true,  priority:160 };
pub const GAK32:  GaTemplate = GaTemplate { name:"gak32",  arena:32_768,  goroutines:16, channels:8,  can_spawn:true,  priority:140 };
pub const GAK64:  GaTemplate = GaTemplate { name:"gak64",  arena:65_536,  goroutines:32, channels:16, can_spawn:true,  priority:120 };
pub const GAK128: GaTemplate = GaTemplate { name:"gak128", arena:131_072, goroutines:64, channels:32, can_spawn:true,  priority:100 };

// ── MB tier ───────────────────────────────
pub const GAM1:   GaTemplate = GaTemplate { name:"gam1",   arena:1_048_576,   goroutines:32,  channels:16, can_spawn:true, priority:80 };
pub const GAM4:   GaTemplate = GaTemplate { name:"gam4",   arena:4_194_304,   goroutines:64,  channels:32, can_spawn:true, priority:60 };
pub const GAM10:  GaTemplate = GaTemplate { name:"gam10",  arena:10_485_760,  goroutines:64,  channels:32, can_spawn:true, priority:40 };
pub const GAM128: GaTemplate = GaTemplate { name:"gam128", arena:134_217_728, goroutines:128, channels:64, can_spawn:true, priority:20 };

// ── gag tier (nested Rust) ────────────────
pub const GAG1:   GaTemplate = GaTemplate { name:"gag1",  kind:Nested, rust_slots:1,  priority:160 };
pub const GAG4:   GaTemplate = GaTemplate { name:"gag4",  kind:Nested, rust_slots:4,  priority:140 };
pub const GAG16:  GaTemplate = GaTemplate { name:"gag16", kind:Nested, rust_slots:16, priority:120 };

// ── gxN supervisor registry ───────────────
pub const GX_REGISTRY: &[GxEntry] = &[
    GxEntry { id:1, role:"network",  file:"gx1", state:Off },
    GxEntry { id:2, role:"io",       file:"gx2", state:Off },
    GxEntry { id:3, role:"database", file:"gx3", state:Off },
];

// ── orchestration rules ───────────────────
pub const GX_RULES: GxRules = GxRules {
    start_all_on_boot:  false,
    stop_all_on_panic:  true,
    restart_on_fault:   true,
    max_restarts:       3,
    gag_max_depth:      2,
};
```

---

## 8. gxr.rs — The Runner

Ephemeral coordinator. Reads gxo. Coordinates gxN. Steps back.

```rust
// gxr.rs — ephemeral runner

fn main() {
    match gxr::command() {
        Start(id)   => gxr::start(id),
        Stop(id)    => gxr::stop(id),
        Restart(id) => gxr::restart(id),
        StartAll    => gxr::start_all(),
        StopAll     => gxr::stop_all(),
        Status      => gxr::status(),
    }
    // done — exits cleanly
}
```

### CLI usage

```bash
gxr start gx1         # start one supervisor
gxr start gx2         # start another
gxr stop gx1          # stop one
gxr restart gx2       # restart one
gxr start --all       # start all registered gxN
gxr stop --all        # graceful full shutdown
gxr status            # show all gxN states + worker counts
```

---

## 9. Go Module — gx-modname

```
gx-modname/
├── gx/
│   ├── hatch.go      — worker hatch entrypoint
│   ├── arena.go      — arena-bounded memory ops
│   ├── channel.go    — channel mgmt within budget
│   ├── signal.go     — done/panic signaling to Rust
│   ├── recover.go    — panic recovery at boundary
│   └── nested.go     — gag nested Rust call bridge
└── go.mod            → module github.com/org/myapp-gx
```

### Hatch entrypoint

```go
// gx/hatch.go

func Hatch(arenaPtr uintptr, arenaSize int, fn func(*Arena)) {
    arena := newArena(arenaPtr, arenaSize)

    defer func() {
        if r := recover(); r != nil {
            SignalPanic(r)  // caught at boundary, never escapes
        }
    }()

    fn(arena)
    SignalDone()
}
```

### Worker example

```go
// user code

gaps.Hatch(func(arena *gx.Arena) {
    ch := arena.NewChannel(4)     // within gak16 budget

    go func() { ch <- fetchData() }()  // goroutine within budget
    go func() { ch <- fetchMeta() }()

    result := <-ch
    meta   := <-ch

    arena.Write(result, meta)     // write to pinned arena
    // SignalDone called automatically
})
```

### Nested Rust call (gag)

```go
// gx/nested.go — calling back into Rust from Go

func CallRust(slots int, fn func() []byte) []byte {
    // signals Rust to execute fn in Rust land
    // blocks goroutine until Rust returns
    // result copied into arena
    return gx_rust_call(slots, fn)
}

// usage in a hatch:
result := gx.CallRust(gag4, func() []byte {
    return rustCrypto(data)  // heavy crypto in Rust
})
```

---

## 10. Rust Crate — gxgr

```
gxgr/
├── src/
│   ├── lib.rs          — public API
│   ├── gxo.rs          — registry (templates, rules)
│   ├── gxr.rs          — runner (start/stop/status)
│   ├── node.rs         — GapNode supervision tree
│   ├── arena.rs        — pinned arena pool
│   ├── reaper.rs       — timeout + budget enforcement
│   ├── bridge_out.rs   — Rust→Go (unsafe #1)
│   └── bridge_in.rs    — Go→Rust gag (unsafe #2)
├── Cargo.toml
└── build.rs
```

### The 2 unsafe blocks — total

```rust
// bridge_out.rs — unsafe #1 — Rust calls Go hatch
// THIS IS UNSAFE BLOCK 1 OF 2 IN THE ENTIRE FRAMEWORK

unsafe extern "C" {
    fn gx_go_hatch(arena_ptr: *mut u8, arena_size: usize, fn_id: u64) -> i32;
}

pub(crate) unsafe fn call_hatch(ptr: *mut u8, size: usize, fn_id: u64) -> i32 {
    gx_go_hatch(ptr, size, fn_id)
}
```

```rust
// bridge_in.rs — unsafe #2 — Go calls back into Rust (gag)
// THIS IS UNSAFE BLOCK 2 OF 2 IN THE ENTIRE FRAMEWORK

#[no_mangle]
pub unsafe extern "C" fn gx_rust_call(slots: u8, fn_id: u64) -> *mut u8 {
    // execute the registered Rust fn
    // return result pointer to Go
    execute_rust_slot(slots, fn_id)
}
```

```
Tokio:   ~100-200 unsafe blocks, scattered everywhere
gxgr:    2 unsafe blocks, both in bridge files, auditable in 5 min 🎯
```

---

## 11. Toolchain — zig cc everywhere

gxgr uses **zig cc** as the C compiler for BOTH dev and release.
This means dev and prod are identical — no glibc surprises at deploy time.

```bash
# install zig (itself a single static binary — no package manager needed)
wget https://ziglang.org/download/0.13.0/zig-linux-x86_64-0.13.0.tar.xz
tar xf zig-*.tar.xz && export PATH=$PATH:~/zig-*
```

### Makefile

```makefile
ZIG_CC  = CC="zig cc" CXX="zig c++" AR="zig ar"
GO_BASE = CGO_ENABLED=1 GOOS=linux GOARCH=amd64
MUSL    = --target x86_64-unknown-linux-musl

dev:
	$(ZIG_CC) $(GO_BASE) go run ./gx/... &
	cargo run $(MUSL)

release:
	$(ZIG_CC) $(GO_BASE) \
	go build -buildmode=c-archive -o target/gx.a ./gx/...
	RUSTFLAGS="-L target/ -l static=gx" \
	cargo build $(MUSL) --release

cross-arm:
	CC="zig cc -target aarch64-linux-musl" \
	$(GO_BASE) GOARCH=arm64 \
	go build -buildmode=c-archive -o target/gx-arm.a ./gx/...

test:
	$(ZIG_CC) $(GO_BASE) go test ./gx/...
	cargo test $(MUSL)
```

### Compile pipeline

```
gx1.go + gx2.go
    ↓ go build -buildmode=c-archive
    gx.a  (Go static lib, C-compatible)

gxo.rs + gxr.rs + gxN.rs
    ↓ rustc --target musl
    + links gx.a via CGo bridge
    ↓ zig cc (musl C compiler)
    ↓ static linker
    myapp  ← single static binary 🎯

$ ldd myapp
not a dynamic executable  ← zero external deps
```

### Binary contents

```
myapp
├── Rust code     (gxo, gxr, gx1, gx2...)
├── Go runtime    (scheduler, GC, goroutine mgmt)
├── Go hatches    (gx1.go, gx2.go...)
├── musl libc     (static)
└── CGo bridge    (the 2 handshake points)

size: ~8-15MB stripped
deps: none
runs: anywhere Linux (bare metal, scratch container, embedded)
```

---

## 12. Comparison

| | Tokio | Smol | gxgr |
|---|---|---|---|
| Language | Pure Rust | Pure Rust | Rust + Go |
| Unsafe in user code | some | some | **zero** |
| Unsafe total | ~100-200 | ~50-100 | **2** |
| Async model | Rust Futures/Wakers | Rust Futures | Go goroutines |
| Scheduler | custom M:N Rust | custom M:N Rust | **Go's proven M:N** |
| Supervision tree | external crate | external crate | **built-in gxo** |
| Budget enforcement | none | none | **ga templates** |
| Multi-supervisor | one runtime | one runtime | **gx1..gxN independent** |
| Turn on/off | all or nothing | all or nothing | **gxr per-supervisor** |
| Panic containment | per-task | per-task | **hatch boundary** |
| Nested Rust calls | N/A | N/A | **gag tier** |
| Static binary | yes (musl) | yes (musl) | **yes (musl+zig)** |
| Dev = Prod toolchain | no | no | **yes (zig cc both)** |
| Cross-compile | complex | complex | **zig cc trivial** |

---

## 13. ga Template Quick Reference

```
gab1    1B      watchdog ping, single flag
gab4    4B      u32 signal
gab64   64B     small probe
gab128  128B    healthcheck
gab256  256B    event watcher
gab512  512B    status reporter
─────────────────────────────────
gak4    4KB     minimal worker
gak8    8KB     lightweight worker
gak16   16KB    standard worker ← sweet spot
gak32   32KB    heavy worker
gak64   64KB    data pipeline
gak128  128KB   large workload
─────────────────────────────────
gam1    1MB     large file chunk  ⚠️ think first
gam4    4MB     heavy streaming   ⚠️ think first
gam10   10MB    🚨 chunk instead
gam128  128MB   💀 reconsider
─────────────────────────────────
gag1    nested  Go→Rust callback (1 slot)
gag4    nested  Go→Rust callback (4 slots)
gag16   nested  Go→Rust callback (16 slots)
```

---

## 14. Publishing

```
crates.io:    gxgr
go mod:       github.com/org/gx-myapp
docs:         docs.rs/gxgr

community naming convention:
→ any Go module for gxgr framework: prefix with gx-
→ gx-network, gx-io, gx-db, gx-yourproject
→ search "gx-" on pkg.go.dev finds them all
```
