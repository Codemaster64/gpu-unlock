# RTX 3080 Ti on Windows: measured unlock, incomplete acceptance

Test date: 5 November 2026. One system, one manual transaction attempt. **No claim of validated daily use, damage-free operation or an unlocked LLM speedup.**

## Configuration

| Component | Observed configuration |
| --- | --- |
| Target | Gigabyte RTX 3080 Ti, 12 GB, GA102; PCI device `10de:2208`, subsystem `1458:4081` |
| Target VBIOS / driver | `94.02.71.40.45` / Windows NVIDIA `610.88` |
| Secondary | RTX A4000 16 GB; alternate monitor connection verified before firmware testing |
| CPU / board | Ryzen 7 9800X3D / MSI PRO X870-P WIFI (MS-7E47), AMI 1.A76 |
| OS | Windows 11 25H2, build 26200.9711; UEFI |
| Power supply | Owner-reported Phanteks AMP GH 1000 W Platinum ATX; connectors checked by owner |
| Matched comparison limits | Target 400 W; secondary temporarily 110 W, down from its normal 140 W |
| Compute runtime | PyTorch 2.11.0+cu128, CUDA runtime 12.8; isolated Python environment |
| Monitoring | Portable HWMonitor 1.58, nvidia-smi, later direct read-only NVML sampling |
| Tested boot implementation | nv-uefi-unlocker v2.1.10, commit `fbdcc5b6e7974864d867f8c878813257b7ab7ca5` |

Secure Boot was already off and the system volume was unencrypted. Neither setting was changed for the experiment. The 100 MB internal EFI system partition was left untouched. These are observations about this machine, not instructions to disable security on another computer.

## Result

8192 x 8192 dense matrix multiplication, TFLOPS, medians of three measured rounds:

| Precision | Stock | Unlocked, incomplete | Recovered stock | Completed unlocked ratio |
| --- | ---: | ---: | ---: | ---: |
| FP16, FP32 accumulation | 67.6475 | 126.1282 | 67.4859 | 1.8645x |
| BF16, FP32 accumulation | 67.5900 | 128.7028 | 69.3529 | 1.9042x |
| TF32 | 34.6635 | not completed | 34.5002 | unavailable |
| Plain FP32 control | 23.7146 | not completed | 24.1419 | unavailable |

At 4096, the completed FP16/BF16 comparisons were about 2.14x. Raw rounds and small correctness-control hashes are in [matrix-results.json](data/matrix-results.json). Stock and recovered suites completed all eight precision/size cases. The unlocked suite completed four cases before stopping.

Correctness controls used deterministic 64-square integer matrices, a CPU double-precision reference and exact output equality. Large timed matrices were checked for finite outputs, **not** against a full numerical reference. Matching small hashes are limited evidence; they do not prove all large matrices or LLM outputs are identical.

The returned stock FP16 rate was within 0.3% of its original baseline after full power removal. This demonstrates the observed rollback on this machine.

## What the firmware log established

The original Linux implementation in this repository was not executed on Windows. A separate read-only UEFI build was prepared from the pinned derivative; 159 host tests passed and its boot successfully enumerated both GPUs and handed back to Windows. Read-only success did not establish transaction safety.

The subsequent manual transaction log recorded one selector-matched target and one attempt. It recorded PLM `ffffff8f` to `ffffffff`, SS0 `88888888`, SS1 `8`, survival across FLR, zero feature handoff readout, DMA release, native GOP display recovery and Windows Boot Manager handoff. The A4000 was not selected for the transaction. Both NVIDIA devices subsequently reported Device Manager error code 0.

The log's early M0 read-only label was a stale unconditional string in the pinned source; build identity and subsequent transaction stages established the actual mode. An empty/unreadable optional GOP-marker message also appeared with the marker absent; normal native display recovery completed. These were source-reviewed observations, not permission to ignore arbitrary future warnings.

The actual transaction EFI SHA-256 was `fd8cc71648e45337c57ebaa652ac14fc92efb8335c283aef67736796b77ce755`. The locally built read-only EFI had a different recorded hash; it is not a universally reproducible binary identifier. See [artifact-manifest.json](data/artifact-manifest.json). Hash agreement establishes artifact identity, not hardware safety.

## Why testing stopped

The unlocked benchmark detected active thermal limiting before TF32 and plain FP32 controls. Its original exception did not preserve the offending GPU or field, so those cannot be attributed retrospectively. The process stopped; no unlocked LLM test, repeat transaction or sustained load followed in that powered session.

HWMonitor recorded target maxima of 52.4 C core, 54 C memory and 62 C hotspot, with Temperature Limit maximum 1. Subsequent nvidia-smi thermal flags were inactive and its software thermal counter equalled the test's initial value. These readings neither explain the earlier flag nor prove overheating or harmlessness. A cool core alone cannot exclude memory-related limiting. NVIDIA documents the event reasons in its [NVML reference](https://docs.nvidia.com/deploy/nvml-api/latest/api/group__nvmlClocksEventReasons.html).

A preliminary stock inference run at the A4000's original 140 W setting had earlier reached our deliberately conservative 80 C core cutoff. The secondary's reported target temperature was 90 C; crossing our cutoff was not itself proof of vendor thermal throttling. That forced server termination coincided with two nvlddmkm Event 153 entries. Causation was not established. Complete subsequent stock/recovered inference comparisons at 110 W had no recurrence of the relevant driver/display/WHEA events during the observed window.

HWMonitor's PCIe PEX recovery count increased during workloads. Other displayed fatal/nonfatal/CRC/replay counters were zero. This was retained as an uncertainty rather than treated as either a proved hardware fault or a clean bill of health.

## Later stock-state telemetry investigation

After rollback, a 180-second passive direct-NVML capture sampled 1,789 raw event masks per GPU. The target masks were all idle (`0x1`); core temperature was 44–47 C. Nevertheless, field 269, the software thermal counter, increased by about 26.7001 seconds. Legacy thermal field 75 and hardware thermal field 270 stayed zero.

A separate synchronized stock matrix run completed all eight cases with the full 7.209-second workload inside the 25-second observation window. Target masks: 1,501 idle (`0x1`), 393 unlimited (`0x0`), 537 power-cap (`0x4`), and no sampled software/hardware thermal bits. Sampled target core maximum was 67 C. HWMonitor had already recorded a Temperature Limit maximum of 1 during ordinary stock desktop use before this matrix run.

NVML memory-temperature field 82 returned `NVML_ERROR_NOT_SUPPORTED` (3) on both cards. An unavailable temperature is not zero. Sampling can miss brief events, and different APIs/counters need not have identical timing semantics. These observations establish a telemetry discrepancy requiring investigation; they do not prove that the original active flag was harmless. [Sanitized observation summaries](data/nvml-observations.json) retain the counts and initial/final values without hardware UUIDs or absolute timestamps.

The first attempted fast trace expired before its matrix job began; it is explicitly excluded from correlated evidence. Only the later synchronized run supports the overlap statement above.

## LLM workload and limits of inference

Although the frontend was described as OpenCode/Ollama, the observed benchmark engine was llama.cpp b12032 behind llama-swap. The model was a Qwen-labelled 27B Q5_K_M GGUF, 19,535,941,024 bytes. The model binary was not redistributed, and the label was not independently audited as an architecture claim.

Comparison settings: context 98,304; full GPU-layer offload request; flash attention on; one parallel slot; Q8 K/V and draft K/V caches; tensor split 9:13 with CUDA order **3080 Ti first, A4000 second**; `draft-mtp,ngram-mod` speculation; maximum draft length 3; reasoning budget 4,096. The exact model/tokenizer and runtime build matter when attempting reproduction. A source revision and full-file weight hash were not recorded, so the model description and byte size alone cannot establish an identical reproduction of the weights.

Two identical synthetic prompt cases were used: 33 and 4,876 prompt tokens. The exact [synthetic fixtures](reference/llm-fixtures.json) are included with their original byte hash. Each had two warmups and three measured requests, fixed seed 42, temperature 0, 64 generated tokens, EOS ignored and prompt caching disabled (`cache_n=0` verified). Output hashes matched within each case. Adaptive ngram speculation can still benefit from repeated prompts even with prompt caching disabled, so short-prompt throughput is not representative of unseen coding work.

| Long synthetic prompt | Stock | Recovered stock |
| --- | ---: | ---: |
| Prompt processing, median tokens/s | 457.71 | 460.09 |
| Generation, median tokens/s | 65.00 | 64.46 |
| Request wall time, median seconds | 11.6261 | 11.5789 |

There is no unlocked LLM result. All these comparisons used the temporary 110 W secondary limit. The GPU tensor-math ratios do not establish any improvement to Q5 generation, prompt processing, games or application latency.

## Host-library crash and diagnostic improvements

The original host process failed in cuBLASLt with CPU exception `0xc000001d` at 4096-square FP16. A paired reproduction passed with the documented, process-local `CUBLASLT_DISABLE_CPU_INSTRUCTIONS_MASK=1`. The same setting was pinned across comparisons. It changes a CPU-library path; it is not a GPU unlock or system-wide setting. See [NVIDIA cuBLAS documentation](https://docs.nvidia.com/cuda/cublas/index.html#disabling-cpu-instructions).

Later diagnostic revisions retained the exact GPU/field/full snapshot on thermal failure. A supervisor stops its own benchmark child if the observer exits, reports an error, loses its byte-progress heartbeat for more than three seconds or exceeds the run deadline. Required sensor/counter failures and stale timestamps fail closed. Twelve CPU-only guard tests passed. **These revised guards have not been validated by another unlocked hardware run.**

The supervisor's checks run about every 50 ms plus OS scheduling latency; termination is not instantaneous. An unresponsive observer allows up to the three-second heartbeat threshold. NVML cannot supply our memory/hotspot readings, so HWMonitor supervision remains necessary. An observer exiting by itself does not terminate an unrelated game or model process.

## Recovery and final state

After the stopped unlocked run, the owner shut down, removed PSU power and the USB, and booted normal Windows. Stock compute and matched inference checks passed. Recovered HWMonitor maxima were 66.5 C core, 64 C memory and 79.5 C hotspot, with Temperature Limit maximum 0 at that stage; the later stock observations are described separately above.

The A4000 was then restored to its original 140 W limit and the target remained at 400 W. Those final settings were checked at idle; no matched inference comparison at 140 W is claimed. Reboot had previously restored target 450 W and secondary 140 W even with tuning applications absent, so the checked limits were not assumed persistent. The reason for the target's 450 W boot setting was not established.

Afterburner and Gigabyte Control Center automatic tuning were disabled for controlled testing, with profiles/task definitions backed up privately. No VBIOS flash, system ESP/BCD/BootOrder modification, ssdiag physical-memory driver or unattended unlock marker was used. Task-created workload processes exited.

Full power removal cleared the measured unlock state in this experiment. A future activation would be another firmware transaction and requires fresh checks. There is no installed daily-use solution in this report.
