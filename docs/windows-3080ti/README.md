# Windows 11 RTX 3080 Ti experiment

**Status: firmware transaction succeeded; workload acceptance incomplete. Daily unlocked operation is not validated.**

On 5 November 2026, one Gigabyte RTX 3080 Ti 12 GB was tested with a separate, version-pinned UEFI implementation. Completed 8192-square FP16/BF16 matrix tests improved about 1.86x/1.90x, then a thermal flag stopped the suite. No unlocked LLM benchmark or sustained stability run was completed. Cold-power rollback and subsequent stock compute/inference checks passed.

## Our before-and-after measurements

8192 x 8192 matrix multiplication, TFLOPS; medians of three measured rounds, with matched 400 W target / 110 W secondary limits:

| Precision | Before (stock) | After (unlocked, incomplete suite) | Gain | After rollback (stock) |
| --- | ---: | ---: | ---: | ---: |
| FP16, FP32 accumulation | 67.65 | 126.13 | +86.4% | 67.49 |
| BF16, FP32 accumulation | 67.59 | 128.70 | +90.4% | 69.35 |
| TF32 | 34.66 | not completed | unavailable | 34.50 |
| Plain FP32 control | 23.71 | not completed | unavailable | 24.14 |

These are matrix-throughput measurements, not an LLM speedup. The thermal stop leaves workload acceptance incomplete. [Raw measured rounds](data/matrix-results.json) and the [full report](REPORT.md#result) document the conditions and limitations.

## Files and implementation

This fork adds the experiment record to [Zagitta/gpu-unlock](https://github.com/Zagitta/gpu-unlock). **It does not port the Linux module to Windows.** The actual Windows boot experiment used [physicsdolphin/nv-uefi-unlocker v2.1.10](https://github.com/physicsdolphin/nv-uefi-unlocker/releases/tag/v2.1.10), source commit `fbdcc5b6e7974864d867f8c878813257b7ab7ca5`.

On our system, external FAT32 USB media ran the UEFI program before Windows, with manual confirmation and a checked target selector. It then handed off to ordinary Windows Boot Manager and the stock NVIDIA driver. We first completed a separate read-only boot and matched stock baselines; after the unlocked thermal stop, full GPU power removal and a normal boot returned measured performance to stock. No automatic daily activation was installed. See [how we tested on Windows](../../README.md#how-we-tested-on-windows) for the overview and the reproduction checklist below for the detailed gates.

- [Report: measurements, failures and rollback](REPORT.md)
- [Reproduction checklist and recovery considerations](REPRODUCING.md)
- [Historical diagnostic source and its limitations](reference/README.md)
- [Machine-readable matrix measurements](data/matrix-results.json)
- [Sanitized inference summary](data/inference-results.json)
- [Later stock-state NVML observations](data/nvml-observations.json)
- [Version and runtime-file hashes, without binaries](data/artifact-manifest.json)

The addition contains no EFI binaries, NVIDIA firmware, GPU-writing installer, startup task or automatic boot configuration. Historical scripts are shared as text references, with personal identifiers replaced. Raw firmware logs and local app configurations are not published.

The most useful next contribution would be evidence that explains the active thermal flag while preserving stop conditions, followed by a fully observed, matched workload comparison. A fast matrix result alone does not establish an LLM benefit or long-term reliability.
