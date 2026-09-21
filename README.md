# Qwen3.8-Flash-Next on Strix Halo: 32 to 61 tokens per second

*A [Vitronia](https://github.com/drluoto/flash-next-strix-halo) project.*

> **20 Sep 2026 — the draft head is now Q5_K, not Q8_0.** A lighter head turned out
> to be faster *and* accepted more often. +2.8 % mean, +8.7 % on prose. Also `-lm dio`,
> not `-lm mmap` — mmap cost 9–12 % decode when measured properly.
>
> **Still Vulkan, not ROCm** (17 Sep): ROCm returns wrong answers on this chip without
> telling you. Old ROCm numbers are at the bottom.

This is a project to continuously optimise the Qwen3.8-Flash MoE model for my
Strix Halo. Some of the optimisations may be relevant to other hardware. Presently
this is the best stack I have found out there; some of the dev work is my own and
some is building on community releases. Hope it may be of use for some of you. When
I have time I try to merge upstream.

125B model, one mini PC, 128 GB shared memory, no cloud.

## The stack, end to end

| | |
|---|---|
| Hardware | Ryzen AI Max+ 395, 128 GB shared, Radeon 8060S (gfx1151) |
| Backend | Vulkan / RADV |
| Model | [AgenticRequant Q5K](https://huggingface.co/drluoto/Qwen3.8-Flash-Next-AgenticRequant-Q5K-GGUF) — trunk IQ4_XS→Q5_K, routers F32→Q8_0, experts untouched |
| Draft head | [MTP Q5_K, FR-Spec 65k](https://huggingface.co/drluoto/Qwen3.8-Flash-Next-MTP-GGUF) — 2.70 GB |
| Engine | [`drluoto/llama.cpp`, branch `strix-halo-vulkan`](https://github.com/drluoto/llama.cpp/tree/strix-halo-vulkan) |
| Speculation | `draft-mtp`, `n-max 3`, `p-min 0.0` |
| Loading | `-lm dio` |
| Context | `-c 262144 -np 2` (131k per slot) |
| GPU clock | pinned `high` |

## Speed

One request at a time, temp 0, no prompt cache.

| Workload | tok/s |
|---|---:|
| short code, empty context | 61 |
| new code @8k | 43 |
| prose @8k | 32 |
| file rewrite @8k | 56 |
| new code @32k | 38 |
| file rewrite @32k | 49 |

Prompt reading: 510 tok/s at 8k, 390 at 32k.

Prose is the floor. The model can't guess its own prose, so speculation buys nothing
there. File rewrite is the ceiling because the output is mostly already in the input.
Turn the draft depth up and rewrite hits 63, but we don't run that daily.

## What got us here

The draft head is most of it. The model ships with a small head that predicts the next
few tokens; the big model verifies them in one pass. We cut its vocabulary to the 65k
tokens our work actually uses.

Then we made it lighter still. Going from Q8_0 to Q5_K sped the whole stack up 2.8 %
on average and 8.7 % on prose — and acceptance went *up*, 0.35 to 0.39 on prose. A
cheaper guess beats a better one when the target verifies it anyway. Credit to
[KYmidnight](https://github.com/ggml-org/llama.cpp/discussions/27950) for pointing
that out.

Session affinity was the surprise. Pinning a conversation to one slot took cold start
from 44 seconds to under one.

The rest is kernel work, some ours, some picked up from others on this chip. One of
ours is [merged upstream](https://github.com/ggml-org/llama.cpp/pull/28501) — raising
the hoisted row-id limit so 512-expert models stop falling back to the scan path.

## Don't bother with

**ROCm.** Wrong logits for any prompt longer than the batch size, silently. Four open
bugs. Perplexity 84.8 against Vulkan's 13.8 on identical settings.

**N-gram drafting.** Each verification costs 230–480 ms on Vulkan. Loses on everything
except prose, where it breaks even.

**Quantised KV cache.** Saves a gigabyte, wrecks tool calling. You have 128 GB.

**A bigger TTM page pool.** We set it near the size of RAM. The driver then holds onto
pages the kernel needs and the machine deadlocks in `drm_suballoc_new`, unkillable,
reboot only. Cost us four reboots before we found it. Leave it at the default.

## Setup

Ryzen AI Max+ 395, 128 GB, Radeon 8060S.

- **Model:** [AgenticRequant Q5K](https://huggingface.co/drluoto/Qwen3.8-Flash-Next-AgenticRequant-Q5K-GGUF)
  — Qwen3.8-Flash-Next requantised by measured read frequency. The trunk is read in
  full on every token and the experts aren't, so the bits went to the trunk.
  +13 % decode over stock, 1.4 GB smaller, 0.91 % perplexity. Recipe and numbers in
  the repo.
- **Draft head:** [MTP sidecar](https://huggingface.co/drluoto/Qwen3.8-Flash-Next-MTP-GGUF),
  vocabulary trimmed to the 65k tokens real agent traces actually use.
- **Engine:** [`drluoto/llama.cpp`, branch `strix-halo-vulkan`](https://github.com/drluoto/llama.cpp/tree/strix-halo-vulkan).
- **Pin the GPU clock.** `rocm-smi -d 0 --setperflevel high`. On `auto` the clock
  floats 2340–2540 MHz against a 2900 max and you lose about 20 % — it also makes
  every A/B you run meaningless.

```sh
llama-server -m trunk-q5k-00001-of-00003.gguf \
  -md mtp-Qwen3.8-Flash-Next-Q5_K-frspec-65k.gguf \
  --spec-type draft-mtp --spec-draft-n-max 3 --spec-draft-p-min 0.0 \
  -fa 1 -ub 2048 -b 2048 -c 262144 -np 2 -lm dio --jinja
```

**We used to recommend `-lm mmap` here. That was wrong.** The idea was sound — it lets
the kernel reclaim the 29 GB n-gram table under pressure instead of OOM-killing the
server — but measured on a pinned clock with interleaved runs it costs 9–12 % decode,
not zero. The earlier measurement was a single run at a floating clock. Use `-lm dio`
and fix the memory pressure at the kernel parameters instead.

## Old ROCm numbers (Aug 2026)

ROCm 7.1, stock UD-IQ4_XS. Kept for anyone who followed the earlier writeup.

| Workload | no spec | with draft head |
|---|---:|---:|
| file rewrite @8k | 17 | 47 |
| new code @8k | 17 | 32 |
| file rewrite @24k | 15 | 29 |
| new code @24k | 15 | 25 |

---

Built with Claude. A Vitronia project.
