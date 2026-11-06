# Windows 11 RTX 3080 Ti experiment

**Status: automatic USB unlock, full bounded matrix suite and synthetic LLM benchmark passed. Daily stability remains unvalidated; the earlier stock thermal flag is unresolved.**

On 5 November 2026, the initial manual attempt increased tensor throughput but stopped on a thermal flag; cold-power rollback passed. On 6 November, a local custom automatic controller using the same pinned transaction recipe successfully unlocked the card and handed off to Windows. The complete matrix suite and bounded synthetic LLM benchmark then passed. These later passes do not explain the earlier flag or establish sustained daily reliability.

## Our before-and-after measurements

Latest 6 November comparison: 8192 x 8192 matrix multiplication, TFLOPS; medians of three rounds, matched 400 W target / 110 W secondary limits:

| Precision | Before (stock) | After (unlocked) | Measured gain |
| --- | ---: | ---: | ---: |
| FP16, FP32 accumulation | 57.64 | 101.33 | +75.8% |
| BF16, FP32 accumulation | 58.31 | 104.91 | +79.9% |
| TF32 | 29.55 | 54.29 | +83.7% |
| Plain FP32 control | 21.48 | 21.50 | +0.1% |

Both runs completed all eight numerical cases, but **the stock control's overall observer failed on a software thermal flag after matrix completion**. The unlocked suite and its 25-second, approximately 100 Hz raw observation passed, with no thermal samples or thermal-counter increase. The ratios compare completed matrix measurements, not two fully accepted stability baselines. [Exact rounds and limits](data/2026-11-06-unlocked-follow-up.json) · [Full follow-up](REPORT.md#6-november-unlocked-follow-up). The [initial 5 November pair and rollback](REPORT.md#result) remain preserved separately.

## LLM: bounded synthetic comparison

Long synthetic prompt: 4,876 input tokens, 64 generated tokens; two warmups and three measured requests per case. Same recorded llama.cpp b12032 profile, model path/size/mtime, executable hash, fixture hash, context, split, KV and speculation settings; comparison limits 400/110 W.

| Metric | Stock reference, 5 November | Unlocked, 6 November | Observed change |
| --- | ---: | ---: | ---: |
| Prompt processing, tokens/s | 457.71 | 568.86 | +24.3% |
| Generation, tokens/s | 65.00 | 75.51 | +16.2% |
| Request time, seconds | 11.6261 | 9.4194 | 19.0% lower |

**This uses a previous-day stock reference**, so it is an indicative matched-profile comparison, not an isolated causal estimate. Output hashes matched within each case and against its stock reference. Repeated prompts and adaptive speculative decoding limit generalization to unseen coding work. The LLM run used approximately one-second telemetry, not the matrix observer's 100 Hz sampling. No ten-minute sustained check or new cold-power rollback was completed after this attempt.

## 6 November stock control

A fresh stock run after a successful local custom **read-only** USB probe completed all eight matrix cases. Its supervised capture failed when direct NVML reported software thermal limiting **0.37 seconds after the benchmark marked itself complete**. Testing stopped in that stock session. The later cold-boot unlocked comparison is shown above; the failed control remains preserved.

The stock event was on the 3080 Ti: raw mask `0x24`, software thermal active, hardware thermal inactive, sampled core 57 C. HWMonitor 1.59 session maxima were 62.7 C core / 60 C memory / 73.9 C hotspot, with Temperature Limit maximum 0. These observations do not prove harmlessness or actual VRAM overheating. [Full stock follow-up](REPORT.md#6-november-stock-control-follow-up) · [Raw rounds and sanitized event](data/2026-11-06-stock-control.json).

## Files and implementation

This fork adds the experiment record to [Zagitta/gpu-unlock](https://github.com/Zagitta/gpu-unlock). **It does not port the Linux module to Windows.** The actual Windows boot experiment used [physicsdolphin/nv-uefi-unlocker v2.1.10](https://github.com/physicsdolphin/nv-uefi-unlocker/releases/tag/v2.1.10), source commit `fbdcc5b6e7974864d867f8c878813257b7ab7ca5`.

The first attempt used manual confirmation. The later local custom controller added exact board/ROM enrollment and payload checks, a five-second any-key cancellation window, persistent failure lockout and automatic Windows handoff. It ran from external FAT32 USB before the normal NVIDIA driver. Its transaction boot succeeded; its requested failure-shutdown/next-power-on setup path has not been hardware tested. At the end of the bounded tests, automatic USB activation was restored and the current unlock had not been rolled back. This fork publishes the evidence and historical diagnostic references; it does not distribute that custom controller or a generic Windows installer. See [how we tested on Windows](../../README.md#how-we-tested-on-windows).

- [Report: measurements, failures and rollback](REPORT.md)
- [Reproduction checklist and recovery considerations](REPRODUCING.md)
- [Historical diagnostic source and its limitations](reference/README.md)
- [Machine-readable matrix measurements](data/matrix-results.json)
- [Sanitized inference summary](data/inference-results.json)
- [Later stock-state NVML observations](data/nvml-observations.json)
- [Version and runtime-file hashes, without binaries](data/artifact-manifest.json)

The addition contains no EFI binaries, NVIDIA firmware, GPU-writing installer, startup task or automatic boot configuration. Historical scripts are shared as text references, with personal identifiers replaced. Raw firmware logs and local app configurations are not published.

The next useful evidence is an explanation of the stock thermal flag, a clean same-day LLM control, and sustained/cold-rollback validation of the custom attempt. The bounded gains above do not establish long-term reliability.
