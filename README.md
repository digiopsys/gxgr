<p align="center">
  <img src="https://img.shields.io/badge/version-v0.1--concept-orange?style=flat-square&labelColor=0d1117" />
  <img src="https://img.shields.io/badge/unsafe_blocks-2_total-00d4aa?style=flat-square&labelColor=0d1117" />
  <img src="https://img.shields.io/badge/binary-static_musl-a78bfa?style=flat-square&labelColor=0d1117" />
  <img src="https://img.shields.io/badge/toolchain-zig_cc-fbbf24?style=flat-square&labelColor=0d1117" />
  <img src="https://img.shields.io/badge/status-design_concept-38bdf8?style=flat-square&labelColor=0d1117" />
</p>

<h1 align="center">gxgr</h1>
<p align="center"><b>The Go-Rust Framework</b></p>
<p align="center">
  Harness Go's concurrency inside Rust-owned, memory-safe supervision.<br/>
  <b>2 unsafe blocks total. Static binary via musl + zig. Systemd-grade reliability.</b>
</p>

---

## Why gxgr Exists

Modern systems software in pure Rust hits the same wall every time: async concurrency requires implementing Wakers, Pin, task vtables, intrusive linked lists — all requiring `unsafe`. Tokio alone has **~150 unsafe blocks** scattered across its source. The Tokio team even added `#![allow(unsafe_op_in_unsafe_fn)]` just to manage the noise.

**gxgr's thesis:**

> Don't make Rust do what Go already solved in 2009.  
> Don't let Go own what Rust should own.  
> Define a clean enforced contract between them.  
> **2 unsafe blocks. Both in bridge files. Both auditable in 5 minutes.**

---

## Naming & Module Structure

```
gxgr          →  framework name (Go-Rust)
gr            →  umbrella pathway concept

crates.io:    gxgr           (search "gx" finds it)
go mod:       xg-modname     (search "xg-" finds all Go side modules)
```

### The gx / xg split

```
gx  =  Rust side   (Go eXecution supervisor)
xg  =  Go side     (eXecution Go hatch)

mirror of each other — one owns, one runs
```

### File layout

```
gxgr/
├── gxo.rs       →  📋 Registry — templates, rules, gxN definitions
│                   pure static Rust const — zero runtime, zero process
│                   compiles as read-only data into the binary
│
├── gxr.rs       →  🎛️ Runner — ephemeral coordinator
│                   gxr start / stop / restart / status
│                   boots briefly, steps back — not a daemon
│
├── gx1.rs       →  ✋ Rust supervisor 1 (independent hand)
├── xg1.go       →  🐹 Go hatch 1       (paired with gx1)
│
├── gx2.rs       →  ✋ Rust supervisor 2
├── xg2.go       →  🐹 Go hatch 2
│
└── bridge/
    ├── bridge_out.rs   →  unsafe #1 — Rust → Go
    └── bridge_in.rs    →  unsafe #2 — Go → Rust (gag)
```

### Why no singleton

```
traditional singleton:
→ one crash  = everything down
→ one allocator = bottleneck
→ monolithic, hard to scale

gxgr model:
→ gxo  = constitution (can't crash — it's just data)
→ gxr  = ephemeral coordinator (starts, steps back)
→ gxN  = independent hands (each owns its own world)
→ gx1 crashes? gx2 and gx3 keep running
→ gxr restarts gx1 from gxo definition
```

---

## ga — Allocation Template System

Every Go worker hatch is given a **ga template** — a predefined contract defining exactly how much memory and how many goroutines it may use. **Workers never allocate themselves. They request a template from gxo.**

### Naming Rule

```
ga + unit prefix + size = the template name
the name IS the spec — zero ambiguity

gab  →  bytes
gak  →  kilobytes
gam  →  megabytes
gag  →  nested Rust execution context (not memory)
```

### Byte tier — `gab`

| Template | Arena | Goroutines | Channels | Use Case |
|----------|-------|------------|----------|----------|
| `gab1` | 1B | 1 | 1 | single flag / WATCHDOG=1 ping |
| `gab4` | 4B | 1 | 1 | u32 signal with context |
| `gab64` | 64B | 1 | 1 | small status probe |
| `gab128` | 128B | 1 | 1 | healthcheck response |
| `gab256` | 256B | 1 | 1 | event with metadata |
| `gab512` | 512B | 2 | 1 | small message payload |

> `gab` = leaf only. `can_spawn = false`. Too tiny to be parents. Highest priority.

### Kilobyte tier — `gak` ⭐ sweet spot

| Template | Arena | Goroutines | Channels | Use Case |
|----------|-------|------------|----------|----------|
| `gak4` | 4KB | 2 | 1 | minimal worker |
| `gak8` | 8KB | 4 | 2 | lightweight worker |
| `gak16` | 16KB | 8 | 4 | **standard worker — 90% of use cases** |
| `gak32` | 32KB | 16 | 8 | heavy worker |
| `gak64` | 64KB | 32 | 16 | data pipeline |
| `gak128` | 128KB | 64 | 32 | large workload |

### Megabyte tier — `gam` ⚠️

| Template | Arena | Goroutines | Channels | Use Case |
|----------|-------|------------|----------|----------|
| `gam1` | 1MB | 32 | 16 | large file chunk processing |
| `gam4` | 4MB | 64 | 32 | heavy streaming I/O |
| `gam10` | 10MB | 64 | 32 | ⚠️ think twice — chunk instead |
| `gam128` | 128MB | 128 | 64 | 🚨 reconsider architecture |

### gag tier — nested Rust context

| Template | Slots | Use Case |
|----------|-------|----------|
| `gag1` | 1 | Go hatch calls back into Rust |
| `gag4` | 4 | 4 parallel Rust callbacks from Go |
| `gag16` | 16 | heavy nested computation |

> `gag` is **NOT a memory size** — it is a nested Rust execution context called FROM inside a Go hatch, back into Rust. See [Nested Rust Calls](#nested-rust-calls--gag) below.

### Rule: large data stays in Rust

```
Rust native (std::fs, memmap2, serde):    Go hatch (via ga arena):
→ file I/O          ✅                    → concurrent I/O    ✅
→ large reads       ✅                    → network handling  ✅
→ CPU computation   ✅                    → goroutine fan-out ✅
→ cryptography      ✅                    → channel pipelines ✅
→ parsing           ✅                    → event loops       ✅
```

> Large data → Rust native. Go hatches stay lean. `gak` is the real home of gxgr.

---

## Worker Address Convention

Every Go worker hatch has a three-part address:

```
gx{supervisor} : ga{template} gp{parent} gs{step}

gx  =  which supervisor owns this worker
ga  =  allocation template (memory + cycle budget)
gp  =  group parent (who spawned this worker)
gs  =  group step (sequence within parent group)
```

### Examples

```
gx1:gak16gp1gs1   →  supervisor 1, 16KB arena, parent 1, step 1
gx1:gak16gp1gs2   →  supervisor 1, 16KB arena, parent 1, step 2
gx1:gak8gp2gs1    →  supervisor 1, 8KB,  child of gp1, step 1
gx2:gab128gp1gs1  →  supervisor 2, 128B probe, parent 1, step 1
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

## Memory Model — Pinned Arena Protocol

> **Go never owns memory. gxN owns everything.**

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

## Nested Rust Calls — gag

`gag` enables a Go hatch to **call back into Rust** for CPU-heavy work without blocking the goroutine scheduler.

```
normal flow:
gxr → gx1 → Go hatch → work → done

gag nested flow:
gxr → gx1 → Go hatch → gag::call_rust() → Rust computes → Go continues → done
```

```go
// xg1.go — calling back into Rust mid-hatch
result := gx.CallRust(gag4, func() []byte {
    return rustCrypto(data)  // heavy crypto in Rust, not blocking Go
})
```

> Max nesting depth: **2 levels.** Beyond that gxr rejects it at runtime.

---

## gxo.rs — The Registry

Pure static Rust. Zero runtime. Zero process. Compiles as read-only const data.

```rust
// gxo.rs — ga template registry (excerpt)

// byte tier
pub const GAB1:   GaTemplate = GaTemplate { name:"gab1",   arena:1,       goroutines:1,  channels:1,  can_spawn:false, priority:255 };
pub const GAB64:  GaTemplate = GaTemplate { name:"gab64",  arena:64,      goroutines:1,  channels:1,  can_spawn:false, priority:253 };
pub const GAB128: GaTemplate = GaTemplate { name:"gab128", arena:128,     goroutines:1,  channels:1,  can_spawn:false, priority:252 };

// KB tier
pub const GAK8:   GaTemplate = GaTemplate { name:"gak8",   arena:8_192,   goroutines:4,  channels:2,  can_spawn:true,  priority:180 };
pub const GAK16:  GaTemplate = GaTemplate { name:"gak16",  arena:16_384,  goroutines:8,  channels:4,  can_spawn:true,  priority:160 };
pub const GAK32:  GaTemplate = GaTemplate { name:"gak32",  arena:32_768,  goroutines:16, channels:8,  can_spawn:true,  priority:140 };

// gag tier (nested Rust — not memory)
pub const GAG1:   GaTemplate = GaTemplate { name:"gag1",  kind:Nested, rust_slots:1,  priority:160 };
pub const GAG4:   GaTemplate = GaTemplate { name:"gag4",  kind:Nested, rust_slots:4,  priority:140 };

// gxN supervisor registry
pub const GX_REGISTRY: &[GxEntry] = &[
    GxEntry { id:1, role:"network",  file:"gx1", state:Off },
    GxEntry { id:2, role:"io",       file:"gx2", state:Off },
];
```

---

## gxr.rs — The Runner

Ephemeral coordinator. Reads gxo. Coordinates gxN. Steps back.

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

## Go Side — xg

The Go hatch runtime lives in `xg` files, paired with their `gx` Rust supervisor:

```
gx1.rs  ←paired with→  xg1.go
gx2.rs  ←paired with→  xg2.go
```

### Hatch entrypoint

```go
// xg1.go — Go hatch entrypoint

package xg

func Hatch(arenaPtr uintptr, arenaSize int, fn func(*Arena)) {
    arena := newArena(arenaPtr, arenaSize)

    defer func() {
        if r := recover(); r != nil {
            SignalPanic(r)  // caught at boundary — never escapes to Rust
        }
    }()

    fn(arena)
    SignalDone()
}
```

### Worker example

```go
// user code

xg.Hatch(func(arena *xg.Arena) {
    ch := arena.NewChannel(4)            // within gak16 budget

    go func() { ch <- fetchData() }()   // goroutine within budget
    go func() { ch <- fetchMeta() }()

    result := <-ch
    meta   := <-ch

    arena.Write(result, meta)
    // SignalDone called automatically on return
})
```

---

## The 2 Unsafe Blocks

**This is the entire unsafe surface of gxgr. Both in bridge files. Never in user code.**

```rust
// bridge/bridge_out.rs — unsafe #1 — Rust calls Go
// THIS IS UNSAFE BLOCK 1 OF 2

unsafe extern "C" {
    fn gx_go_hatch(ptr: *mut u8, size: usize, fn_id: u64) -> i32;
}

pub(crate) unsafe fn call_hatch(ptr: *mut u8, size: usize, fn_id: u64) -> i32 {
    gx_go_hatch(ptr, size, fn_id)
}
```

```rust
// bridge/bridge_in.rs — unsafe #2 — Go calls back Rust (gag)
// THIS IS UNSAFE BLOCK 2 OF 2

#[no_mangle]
pub unsafe extern "C" fn gx_rust_call(slots: u8, fn_id: u64) -> *mut u8 {
    execute_rust_slot(slots, fn_id)
}
```

| | Tokio | Smol | gxgr |
|---|---|---|---|
| unsafe blocks | ~150 | ~60 | **2** |
| location | scattered everywhere | scattered | **one bridge file** |
| in user code | some | some | **never** |
| auditable in 5 min | ❌ | ❌ | **✅** |

---

## Toolchain — zig cc everywhere

zig cc is used as the C compiler for **both dev and release**. Dev = Prod. No surprises.

```bash
# install zig — itself a single static binary
wget https://ziglang.org/download/0.13.0/zig-linux-x86_64-0.13.0.tar.xz
tar xf zig-*.tar.xz && export PATH=$PATH:~/zig-linux-x86_64-0.13.0
```

### Makefile

```makefile
ZIG_CC  = CC="zig cc" CXX="zig c++" AR="zig ar"
GO_BASE = CGO_ENABLED=1 GOOS=linux GOARCH=amd64
MUSL    = --target x86_64-unknown-linux-musl

dev:
	$(ZIG_CC) $(GO_BASE) go run ./xg/... &
	cargo run $(MUSL)

release:
	$(ZIG_CC) $(GO_BASE) \
	go build -buildmode=c-archive -o target/xg.a ./xg/...
	RUSTFLAGS="-L target/ -l static=xg" \
	cargo build $(MUSL) --release

cross-arm:
	CC="zig cc -target aarch64-linux-musl" \
	$(GO_BASE) GOARCH=arm64 \
	go build -buildmode=c-archive -o target/xg-arm.a ./xg/...

test:
	$(ZIG_CC) $(GO_BASE) go test ./xg/...
	cargo test $(MUSL)
```

### Compile pipeline

```
xg1.go + xg2.go
    ↓  go build -buildmode=c-archive
    xg.a  (Go static lib + Go runtime baked in)

gxo.rs + gxr.rs + gx1.rs + gx2.rs
    ↓  rustc --target musl
    links xg.a via bridge (zig cc as C compiler)
    ↓  static linker (musl)
    myapp  ← single static binary

$ ldd myapp
not a dynamic executable  ← zero external dependencies
```

### Binary contents

```
myapp (~8–15MB stripped)
├── Rust code      gxo + gxr + gx1 + gx2
├── Go runtime     scheduler + GC + goroutine mgmt
├── Go hatches     xg1.go + xg2.go
├── musl libc      static — no glibc
└── CGo bridge     2 unsafe handshakes
```

---

## Comparison

| | Tokio | Smol | gxgr |
|---|---|---|---|
| Language | Pure Rust | Pure Rust | **Rust + Go** |
| unsafe in user code | some | some | **zero** |
| unsafe total | ~150 | ~60 | **2** |
| Async model | Rust Futures/Wakers | Rust Futures | **Go goroutines** |
| Scheduler | custom M:N Rust | custom M:N Rust | **Go's proven M:N** |
| Supervision tree | external crate | external crate | **built-in gxo** |
| Budget enforcement | none | none | **ga templates** |
| Multi-supervisor | one runtime | one runtime | **gx1..gxN independent** |
| Turn on/off | all or nothing | all or nothing | **gxr per-supervisor** |
| Panic containment | per-task | per-task | **hatch boundary** |
| Nested Rust calls | N/A | N/A | **gag tier** |
| Static binary | yes (musl) | yes (musl) | **yes (musl+zig)** |
| Dev = Prod toolchain | ❌ | ❌ | **✅ zig cc both** |
| Cross-compile | complex | complex | **zig cc trivial** |

---

## ga Quick Reference

```
gab1    1B      watchdog ping, single flag
gab4    4B      u32 signal
gab64   64B     small probe
gab128  128B    healthcheck
gab256  256B    event watcher
gab512  512B    status reporter
─────────────────────────────────────────
gak4    4KB     minimal worker
gak8    8KB     lightweight worker
gak16   16KB    standard worker  ← sweet spot ⭐
gak32   32KB    heavy worker
gak64   64KB    data pipeline
gak128  128KB   large workload
─────────────────────────────────────────
gam1    1MB     large file chunk    ⚠️  think first
gam4    4MB     heavy streaming     ⚠️  think first
gam10   10MB                        🚨  chunk instead
gam128  128MB                       💀  reconsider
─────────────────────────────────────────
gag1    nested  Go→Rust callback (1 slot)
gag4    nested  Go→Rust callback (4 slots)
gag16   nested  Go→Rust callback (16 slots)
```

---

## Roadmap

| Phase | Milestone | Status |
|-------|-----------|--------|
| v0.1 | gxo registry, ga templates (all tiers), gxN struct | 🔵 design |
| v0.2 | xg Go hatch entrypoint, bridge_out, SignalDone/Panic | ⬜ pending |
| v0.3 | Supervision tree, reaper, timeout enforcement | ⬜ pending |
| v0.4 | gp/gs hierarchy, child reaping on parent death | ⬜ pending |
| v0.5 | gag nested Rust calls, bridge_in, depth limit | ⬜ pending |
| v0.6 | gxr CLI (start/stop/restart/status) | ⬜ pending |
| v0.7 | zig cc build pipeline, musl static binary | ⬜ pending |
| v0.8 | cycle budget enforcement | ⬜ pending |
| v0.9 | cross-compilation (arm64, riscv64) | ⬜ pending |
| v1.0 | stable API → crates.io/gxgr + go mod xg-\* | ⬜ pending |

---

## Publishing

```
crates.io:    gxgr
go mod:       github.com/org/xg-myapp
docs:         docs.rs/gxgr

community naming convention:
→ Rust crates  prefix with gx-   →  gx-network, gx-io, gx-db
→ Go modules   prefix with xg-   →  xg-network, xg-io, xg-db
→ search "gx-" on crates.io   → finds all Rust side crates
→ search "xg-" on pkg.go.dev  → finds all Go side modules
```

---

## Q&A

### Why use gxgr if I just want async? Why not just use Go?

If you **only need async, just use Go.** gxgr isn't for that.

gxgr is for when you need **both** at the same time:

| You need from Go | You need from Rust |
|------------------|--------------------|
| Concurrent connections | Memory safety without GC |
| Goroutine fan-out | Ownership model |
| Channel pipelines | Compile-time correctness |
| Network event loops | Systems-level control |

Pure Go gives you great async but the GC owns your memory — it can pause, it can't be reasoned about at the systems level. Pure Rust gives you memory safety but async requires ~150 unsafe blocks just to build the runtime. gxgr says: **let Go own the concurrency, let Rust own the memory.** If you only need one of those things — use that one language.

---

### What is gxgr actually designed to build?

Systems-level software where you need both memory control AND high concurrency:

- Init systems (systemd replacement)
- Container runtimes
- Daemon supervisors
- Network proxies
- Security-critical tooling
- Embedded daemons (static binary, runs anywhere Linux)

It is **not** a web framework. It is **not** for REST APIs. It lives close to the OS.

---

### What are gxo, gxr, and gxN?

| File | Role | Runs? |
|------|------|-------|
| `gxo.rs` | 📋 Registry — templates, rules, gxN definitions. Pure static const data. | ❌ Never |
| `gxr.rs` | 🎛️ Runner — starts/stops/restarts gxN. Ephemeral coordinator. | ✅ Briefly, then exits |
| `gxN.rs` | ✋ Supervisor hands — each independently owns its arena pool and worker tree. | ✅ Until stopped |

No singleton. `gx1` can crash without affecting `gx2` or `gx3`. `gxo` is immortal because it never runs — it is just data.

---

### What is the gx / xg split?

```
gx  =  Rust side   — supervisors, memory owners
xg  =  Go side     — hatches, concurrent workers

gx1.rs  ←paired with→  xg1.go
gx2.rs  ←paired with→  xg2.go

see gx = Rust owns it
see xg = Go runs it
```

They rhyme, they mirror, they're unambiguous. `gxgr` on crates.io, `xg-*` on pkg.go.dev.

---

### What is the ga/gp/gs worker address convention?

```
gx{supervisor} : ga{template} gp{parent} gs{step}

gx1:gak16gp1gs1   →  supervisor 1, 16KB arena, parent group 1, step 1
gx1:gab128gp1gs2  →  supervisor 1, 128B probe, parent group 1, step 2
gx2:gak32gp1gs1   →  supervisor 2, 32KB arena, parent group 1, step 1
```

When a parent group is reaped, gxN automatically walks the `gp` tree and reaps all `gs` children.

---

### What is the gag tier — is it actually gigabytes?

No. `gag` is a **nested Rust execution context** called from inside a Go hatch — not a memory size.

```
normal:  gxr → gx1 → Go hatch → work → done
gag:     gxr → gx1 → Go hatch → gag::call_rust() → Rust computes → Go continues → done
```

When a Go hatch needs heavy CPU work mid-flight (crypto, compression, parsing), it calls `gag::call_rust()` — Rust handles the CPU-heavy bit and returns the result to Go. Max nesting depth: **2 levels.**

---

### How many unsafe blocks does gxgr have?

**2 unsafe blocks. Total. Ever.** Both in `bridge/`. Never in user code. Both auditable in 5 minutes.

Compare: Tokio ~150, Smol ~60, gxgr **2**.

---

### Why does Tokio need so many unsafe blocks?

Rust's async model requires unsafe by design at the runtime level — Waker vtables need raw function pointers, task scheduling needs intrusive linked lists, Pin needs `Pin::new_unchecked`, and UnsafeCell is unavoidable in a scheduler. Tokio is brilliant engineering — but you are reimplementing in unsafe Rust what Go already solved safely in 2009. gxgr sidesteps the problem by **never implementing a Rust async runtime at all.**

---

### Can a Go panic escape the hatch and crash the Rust process?

No. Every hatch wraps execution in `defer/recover`. Panics are caught at the boundary, the arena is reclaimed, gxN marks the node `Panicked`. Rust never sees the panic.

```go
defer func() {
    if r := recover(); r != nil {
        SignalPanic(r)  // caught — never escapes to Rust
    }
}()
```

---

### Why zig cc instead of gcc or clang?

zig cc handles musl static targeting trivially, cross-compiles to any target from one install, and is itself a single static binary — no package manager needed. Most importantly: **zig cc is used in both dev and release**, making dev and prod environments identical. No "works on my machine" surprises at deploy time.

---

### How do two languages compile into one binary?

```bash
# Step 1 — Go compiles to C-compatible static archive
CGO_ENABLED=1 go build -buildmode=c-archive -o target/xg.a ./xg/...

# Step 2 — Rust compiles and links the Go archive
RUSTFLAGS="-L target/ -l static=xg" \
cargo build --target x86_64-unknown-linux-musl --release

# Result
$ ldd myapp
not a dynamic executable  ← zero external dependencies
```

One binary. Rust code + Go runtime + Go hatches + musl libc + CGo bridge. ~8–15MB stripped.

---

### Is gxgr production ready?

For production async work today: **Tokio is the answer.** gxgr is the answer when you need zero unsafe + built-in supervision + systems-level memory control — and are willing to be early.

> Use Tokio today. Watch gxgr. Build gxgr. 🔥

---
