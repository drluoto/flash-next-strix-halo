# Qwen3.8-Flash-Next on Strix Halo: 30 to 58 tokens per second

*A [Vitronia](https://github.com/drluoto/flash-next-strix-halo) project.*

> **17 Sep 2026 — this is now a Vulkan stack, not ROCm.** ROCm is faster at reading
> prompts and returns wrong answers on this chip without telling you. Old ROCm
> numbers are at the bottom.

This is a project to continuously optimise the Qwen3.8-Flash MoE model for my
Strix Halo. Some of the optimisations may be relevant to other hardware. Presently
this is the best stack I have found out there; some of the dev work is my own and
some is building on community releases. Hope it may be of use for some of you. When
I have time I try to merge upstream.

125B model, one mini PC, 128 GB shared memory, no cloud.

## Speed

One request at a time, temp 0, no prompt cache.

| Workload | tok/s |
|---|---:|
| short code, empty context | 58 |
| new code @8k | 42 |
| prose @8k | 30 |
| file rewrite @8k | 55 |
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

Session affinity was the surprise. Pinning a conversation to one slot took cold start
from 44 seconds to under one.

The rest is kernel work, some ours, some picked up from others on this chip. One of
ours is [upstream](https://github.com/ggml-org/llama.cpp/pull/28501).

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

Ryzen AI Max+ 395, 128 GB, Radeon 8060S. Qwen3.8-Flash-Next requantised to 86 GB —
the trunk kept at higher precision than stock, since it's read every token and the
experts aren't. Draft head [on Hugging Face](https://huggingface.co/drluoto/Qwen3.8-Flash-Next-MTP-GGUF).
Engine: [`drluoto/llama.cpp`, branch `strix-halo-vulkan`](https://github.com/drluoto/llama.cpp/tree/strix-halo-vulkan).

```sh
llama-server -m trunk-q5k-00001-of-00003.gguf \
  -md mtp-Qwen3.8-Flash-Next-Q8_0-frspec-65k.gguf \
  --spec-type draft-mtp --spec-draft-n-max 3 --spec-draft-p-min 0.0 \
  -fa 1 -ub 2048 -b 2048 -c 262144 -np 2 -lm mmap --jinja
```

`-lm mmap` matters: it lets the kernel reclaim the 29 GB lookup table under pressure
instead of OOM-killing the server. Read the table into cache once after startup and it
costs nothing.

## Old ROCm numbers (Aug 2026)

ROCm 7.1, stock UD-IQ4_XS. Kept for anyone who followed the earlier writeup.

| Workload | no spec | with draft head |
|---|---:|---:|
| file rewrite @8k | 17 | 47 |
| new code @8k | 17 | 32 |
| file rewrite @24k | 15 | 29 |
| new code @24k | 15 | 25 |

---

Built with Claude Fable 5.1. A Vitronia project.
