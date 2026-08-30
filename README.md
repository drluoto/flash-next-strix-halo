# Qwen3.8-Flash-Next on Strix Halo: 17 → 47 tok/s

*A [Vitronia](https://github.com/drluoto) project.*

A working, measured, reproducible stack for running **Qwen3.8-Flash-Next**
(125B-A6B MoE + 51B engram table, `qwen4exp`) on **AMD Ryzen AI Max+ 395 /
Radeon 8060S (gfx1151, 128 GB unified)** with llama.cpp — including the first
working native MTP speculative decoding on ROCm.

Everything here was measured on real coding workloads (file rewrite, targeted
bugfix, new code, prose) at 8k and 24k context, greedy, with output read for
correctness — not just tok/s. Raw threads with full data:
[#26592](https://github.com/ggml-org/llama.cpp/pull/26592),
[#27466](https://github.com/ggml-org/llama.cpp/pull/27466),
[#27836](https://github.com/ggml-org/llama.cpp/pull/27836).

## Results (UD-IQ4_XS, single stream)

| decode tok/s | file rewrite | bugfix | new code | prose |
|---|---|---|---|---|
| no speculation, 8k ctx | 16.8 | 16.3 | 16.8 | 20.7 |
| **full stack, 8k ctx** | **47.1** | 24.7–40 | **31.7** | **24.1** |
| no speculation, 24k ctx | ~15 | | ~15 | |
| **full stack, 24k ctx** | **28.6** | | **25.4** | |

Prefill ~340 tok/s empty, ~200 at 24k. Model + 262k context + vision:
~106 GB of 128. Quality gates: greedy output clean at long prompts, arithmetic
in exact format at 24k depth, tool calling, vision — all verified, plus two
30-minute mixed-load soaks with zero errors.

## The stack

Three composable levers on top of master:

1. **GPU TOP_K** — the QSA sparse-attention indexer runs `ggml_top_k` over the
   whole context, 12×/token. Stock llama.cpp on HIP falls back to **CPU** past
   1024 elements (no CUB), which is why decode collapses with depth (knee at
   exactly d1024). Fix: hipCUB ([#26592](https://github.com/ggml-org/llama.cpp/pull/26592))
   or the native radix kernel ([#27466](https://github.com/ggml-org/llama.cpp/pull/27466)).
   +21 % decode at 16k depth, +38–53 % end-to-end at 24k.
2. **Native MTP** ([#27836](https://github.com/ggml-org/llama.cpp/pull/27836)) —
   the model's own 4B draft head. Needs crusaderky's detached-head loader fix
   and a converter whitelist fix ([rmonsurate/llama.cpp#1](https://github.com/rmonsurate/llama.cpp/pull/1)).
   Output is byte-clean (earlier community MTP ports corrupted above ~1k prompt
   tokens — always read the output, 42 tok/s of noise still reports 42 tok/s).
3. **ngram-mod** — draft from text already in context; free, huge on file
   rewrites. Composes with MTP: `--spec-type draft-mtp,ngram-mod`.

Pre-assembled branch (all of the above, with attribution):
**[`drluoto/llama.cpp` → `strix-halo-flash-next`](https://github.com/drluoto/llama.cpp/tree/strix-halo-flash-next)**

MTP sidecar (converted from the official checkpoint, sha256 in the card):
**[`drluoto/Qwen3.8-Flash-Next-MTP-GGUF`](https://huggingface.co/drluoto/Qwen3.8-Flash-Next-MTP-GGUF)**

## Build (gfx1151, Ubuntu ROCm packages)

```sh
git clone -b strix-halo-flash-next https://github.com/drluoto/llama.cpp
cd llama.cpp
cmake -B build -DGGML_HIP=ON -DAMDGPU_TARGETS=gfx1151 -DGPU_TARGETS=gfx1151 \
  -DGGML_HIP_ROCWMMA_FATTN=ON -DGGML_HIP_MMQ_MFMA=ON -DGGML_HIP_NO_VMM=ON \
  -DCMAKE_BUILD_TYPE=Release
cmake --build build -j --target llama-server
```

`GGML_HIP_NO_VMM=ON` is mandatory on gfx1151 (HIP VMM crashes at graph exec).
If cmake resolves `/opt/rocm` into a broken path, pin
`-Dhip_DIR=/usr/lib/x86_64-linux-gnu/cmake/hip -Damd_comgr_DIR=/usr/lib/x86_64-linux-gnu/cmake/amd_comgr`.

## Run

```sh
HSA_OVERRIDE_GFX_VERSION=11.5.1 GGML_CUDA_DISABLE_GRAPHS=1 \
./build/bin/llama-server \
  -m Qwen3.8-Flash-Next-UD-IQ4_XS-00001-of-00003.gguf \
  -md mtp-Qwen3.8-Flash-Next-Q8_0.gguf \
  --spec-type draft-mtp,ngram-mod --spec-draft-n-max 3 \
  --spec-ngram-mod-n-max 64 --spec-ngram-mod-n-match 24 \
  --mmproj mmproj-F16.gguf \
  -lm dio -fa 1 -ctk f16 -ctv f16 -c 262144 -np 1 --jinja \
  --host 127.0.0.1 --port 8097
```

## Pitfalls we hit so you don't have to

- **`GGML_CUDA_DISABLE_GRAPHS=1` is required on ROCm < 7.13** with the hipCUB
  path: rocPRIM's `DeviceSegmentedRadixSort` is not capture-safe until 4.4.0
  (crashes `operation not permitted when stream is capturing` mid-generation).
  The capture guard lands in ROCm 7.13+ (per Geramy in #26592). HIP graphs are
  worth ~nothing on this model anyway (measured 26.0 vs 27.0 tok/s).
- **Don't tune with `llama-bench` if you run speculation.** It benches without
  speculation; `-ub 2048` won its prefill chart and *halved* real decode
  (22.0 vs 40.9) while eating 12 GB, which made full 262k context look
  impossible when it actually fits.
- **`--load-mode dio`** loads the 93 GB model in ~33 s; mmap takes 3–13 min on
  this platform.
- **Read the output.** The broken MTP fork measured a genuine-looking
  42.7 tok/s while emitting multilingual noise — corruption began between 718
  and 2668 prompt tokens, so every short smoke test passed.
- **Fleet serving is not worth it**: 512 experts × top-10 means 4 parallel
  streams touch up to 40 expert sets — aggregate only 1.36× single-stream, and
  speculation stops helping entirely at `-np 4`. This model is a single-slot
  specialist on this hardware.
- **MTP draft depth:** `n-max 6` with `--spec-draft-p-min 0.7` beats flat `n-max 3`
  on nearly everything (+22% bugfix, +10% prose; credit sammcj). `draft-mtp` *alone*
  beats the combo on bugfix and novel-code-at-depth (ngram drafts displace
  higher-acceptance MTP drafts), while the combo wins echo-heavy file rewrites by
  37% on ROCm — pick per workload; the combo is the better all-rounder for agents.
- Long draft windows (`ngram-mod 256/16`) win file rewrites but drop bugfix
  workloads *below* baseline. `64/24` is the no-regression setting.
- `-ot per_layer_token_embd=CPU` only saves RAM with `-lm mmap` (25× slower
  load); with everything resident it saves nothing. With 128 GB you don't need it.

## Extracting the MTP head yourself

The 4B draft head was cut out of the 360 GB checkpoint with HTTP range reads
(5.2 GB fetched) using **[shard-scalpel](https://github.com/drluoto/shard-scalpel)**,
then converted with #27836's `--mtp` export.

## Changelog

- **2026-08-30:** long-context decode fixes from [#27977](https://github.com/ggml-org/llama.cpp/pull/27977)
  merged (early-exit ngram predecessor scan + gathered QSA decode window + indexer
  kernel shape). File-rewrite @24k: 29.5 → **34.8 tok/s**; depth curve flattens
  (@32k now decodes like @16k did before). Gains grow with context.
- **2026-08-29:** correctness follow-ups from [#27941](https://github.com/ggml-org/llama.cpp/pull/27941)
  merged into the branch (M-RoPE image block scoring — matters if you use vision;
  the `gridDim.y` abort at `n_kv` 262144 — matters at full context; sequence-copy
  indexer keys). Revalidated on gfx1151.

## Credits

The heavy lifting is other people's: **rmonsurate** (MTP implementation),
**crusaderky** (detached-head loader), **Geramy** (hipCUB path + rocPRIM
diagnosis), **jadenmach2** (radix TOP_K), **danielhanchen/Unsloth** (qwen4exp
port + quants), **JJJYmmm/Qwen** (original arch PR). This repo is the
measurement work and the assembly.
