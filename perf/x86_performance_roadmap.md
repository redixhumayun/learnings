# x86-64 Performance Engineering Roadmap

A lean, no-nonsense roadmap for **x86-64 performance engineering**.

## Setup (day 0)
- Get an **x86-64 Linux** box (cloud VM is fine).  
- Install: `gcc g++ clang llvm-mca binutils gdb perf valgrind linux-tools-$(uname -r)`; optional: Intel VTune.
- Use **Intel syntax** everywhere (`objdump -Mintel`).

## 1) ISA Basics (1–2 days)
- Learn registers (GPRs/XMM/YMM/ZMM), flags, addressing modes, `mov/lea/add/sub/imul/shl/shr/and/or/xor/test/cmp/jcc/cmov`.  
- Deliverable: hand-write a tiny `.s` that adds 3 ints and branches; assemble & run.

## 2) ABI & Calling Conventions (1–2 days)
- Read **System V AMD64 ABI**: arg passing, callee/caller-saved regs, stack frame, red zone, varargs.  
- Exercise: write a C/Rust function, compile `-O3`, **disassemble** (`objdump -drwC -Mintel a.out`) and explain every instruction.  
- Confirm prologue/epilogue and where args/returns live.

## 3) Toolchain Fluency (2–3 days)
- **Compiler Explorer**: correlate source ↔ asm at `-O0/-O2/-O3`.  
- Local: `gcc/clang -O3 -S -fverbose-asm -masm=intel foo.c`, `rustc -O --emit asm foo.rs`.  
- Profiling essentials:  
  - `perf stat -e cycles,instructions,branches,branch-misses,cache-misses ./prog`  
  - `perf record -g ./prog && perf report`  
  - `perf annotate` → read the hot loop in asm.

## 4) Microarchitecture Mental Model (2–3 days)
- Pipelines, µops, ports, rename, dep chains, fusion, branch prediction basics.  
- Latency vs throughput; code size & I-cache; alignment.  
- Tool: **llvm-mca** on a hot loop (`llvm-mca -mcpu=skylake -timeline foo.s`); spot bottleneck ports and deps.  
- Deliverable: shorten a dependency chain and show reduced cycles.

## 5) Memory & Caches (3–4 days)
- Lines, sets, associativity, TLBs, prefetchers, write-allocate, streaming stores.  
- False sharing, alignment, `prefetcht0..nta`, `pause`, `lfence/sfence/mfence`, `lock`ed ops.  
- Exercise: write a microbench that walks arrays with strides; plot throughput vs stride; explain L1/L2/LLC cliffs.

## 6) SIMD for Real Work (4–5 days)
- **SSE2 baseline → AVX2 first** (AVX-512 later; know it can downclock).  
- Learn intrinsics (Intel Intrinsics Guide) for loads/stores, permutes, masks, reductions.  
- Exercise: implement dot product 3 ways (scalar, SSE2, AVX2). Compare with `-O3` auto-vec.  
- Validate with `perf stat` (IPC, cycles) and check for spill/misalignment in `perf annotate`.

## 7) Branches & Branchless (1–2 days)
- Patterns: `cmov`, `setcc`, bit-tricks, LUTs, predication via masks; avoid unpredictable branches.  
- Exercise: rewrite a branchy hot path branchless; prove drop in `branch-misses` and cycles.

## 8) Real-World Tuning Loop (ongoing)
- Pick a real kernel (parser, hash, memcmp/memcpy variant, compression inner loop, column scan in a toy OLAP operator).  
- Cycle: **measure → read asm → form a microarch hypothesis → change → remeasure**.  
- Deliverable: **≥10–20%** speedup over compiler baseline or a clear “can’t improve” with evidence.

## 9) Concurrency & System Effects (2–3 days)
- Atomics and memory ordering on x86 (TSO), `lock xadd/cmpxchg`, `pause`, backoff.  
- NUMA, huge pages, page faults, syscall costs; `numactl`, `perf c2c`.  
- Exercise: demonstrate false sharing regression and fix it.

## 10) Advanced/Optional (as needed)
- **AVX-512** (mask regs, compress/expand, gather/scatter)—use when it doesn’t tank frequency.  
- RTM/TSX (if HW supports), PEBS/LBR for deeper profiling, `vtune` for front-end vs back-end stalls.

---

### Weekly cadence
- **Mon:** learn a concept (1–2 hrs) + tiny asm exercise.  
- **Tue–Wed:** one **microbench**; model with `llvm-mca`; verify with perf.  
- **Thu–Fri:** apply to a **real hot path**; ship a measured win or a write-up.

### Checkpoints
- [ ] Read `perf report` and jump straight to the hot loop’s asm.  
- [ ] Explain every instruction in a nontrivial function and why it’s there.  
- [ ] Use `llvm-mca` to predict a loop’s bottleneck and confirm with perf.  
- [ ] Produce ≥3 real workload wins ≥10%.  
- [ ] Know when **not** to hand-write asm.

### References
- **AMD64 System V ABI** (calling convention).  
- **Intel® 64/IA-32 SDM** (Vol. 2 for instructions).  
- **Agner Fog** microarch + optimization guides.  
- **Intel Intrinsics Guide**.  
- **Compiler Explorer**.

### Common traps
- Chasing **microbench mirages**.  
- Ignoring **code size** and I-cache.  
- Blind **AVX-512** use.  
- Hand-coding asm when **intrinsics/data layout** wins more.  
- Forgetting **ABI** details (red zone, callee-saved regs).
