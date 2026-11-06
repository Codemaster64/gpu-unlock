# RTX 3080 Ti on Windows: bounded gains, incomplete daily acceptance

Tests: 5–6 November 2026 on one system. The initial manual attempt stopped on a thermal flag; a later local custom automatic attempt passed the full bounded matrix and synthetic LLM tests. **The stock thermal discrepancy remains unresolved, and sustained daily use is not validated.** Latest results are in the [6 November unlocked follow-up](#6-november-unlocked-follow-up); earlier stages remain below with their original outcomes.

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

Initial 5 November result: 8192 x 8192 dense matrix multiplication, TFLOPS, medians of three rounds. This historical partial attempt is separate from the later full suite below.

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

The initial 5 November unlocked benchmark detected active thermal limiting before TF32 and plain FP32 controls. Its original exception did not preserve the offending GPU or field, so those cannot be attributed retrospectively. The process stopped; no unlocked LLM test, repeat transaction or sustained load followed in that powered session.

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

There was no unlocked LLM result on 5 November. Those stock/recovered comparisons used the temporary 110 W secondary limit. The later bounded unlocked LLM comparison is reported below; matrix ratios alone still do not establish an application speedup.

## Host-library crash and diagnostic improvements

The original host process failed in cuBLASLt with CPU exception `0xc000001d` at 4096-square FP16. A paired reproduction passed with the documented, process-local `CUBLASLT_DISABLE_CPU_INSTRUCTIONS_MASK=1`. The same setting was pinned across comparisons. It changes a CPU-library path; it is not a GPU unlock or system-wide setting. See [NVIDIA cuBLAS documentation](https://docs.nvidia.com/cuda/cublas/index.html#disabling-cpu-instructions).

Later diagnostic revisions retained the exact GPU/field/full snapshot on thermal failure. A supervisor stops its own benchmark child if the observer exits, reports an error, loses its byte-progress heartbeat for more than three seconds or exceeds the run deadline. Required sensor/counter failures and stale timestamps fail closed. Twelve CPU-only guard tests passed. The revised matrix supervisor was used in the successful 6 November unlocked run; this is bounded evidence, not daily reliability certification.

The supervisor's checks run about every 50 ms plus OS scheduling latency; termination is not instantaneous. An unresponsive observer allows up to the three-second heartbeat threshold. NVML cannot supply our memory/hotspot readings, so HWMonitor supervision remains necessary. An observer exiting by itself does not terminate an unrelated game or model process.

## 5 November recovery and operating state

After the stopped unlocked run, the owner shut down, removed PSU power and the USB, and booted normal Windows. Stock compute and matched inference checks passed. Recovered HWMonitor maxima were 66.5 C core, 64 C memory and 79.5 C hotspot, with Temperature Limit maximum 0 at that stage; the later stock observations are described separately above.

The A4000 was then restored to its original 140 W limit and the target remained at 400 W. Those final settings were checked at idle; no matched inference comparison at 140 W is claimed. Reboot had previously restored target 450 W and secondary 140 W even with tuning applications absent, so the checked limits were not assumed persistent. The reason for the target's 450 W boot setting was not established.

Afterburner and Gigabyte Control Center automatic tuning were disabled for controlled testing, with profiles/task definitions backed up privately. No VBIOS flash, system ESP/BCD/BootOrder modification, ssdiag physical-memory driver or unattended unlock marker was used. Task-created workload processes exited.

Full power removal cleared that initial unlock state. As of that recovery, no daily-use solution was installed. The subsequent custom activation and current unrolled-back state are recorded below; this historical recovery is not proof of rollback after the later attempt.

## 6 November stock-control follow-up

At the earlier 6 November control stage, a local custom controller compiled without the transaction feature successfully booted from USB, verified the target and seven pinned assets, recorded a firmware-shadow ROM fingerprint and handed off to Windows. Its log explicitly established non-TX mode. Windows Boot Manager StartImage was logged at 3.573 seconds from EFI entry; this is not desktop-ready boot time. The board advertised support for next-power-on firmware setup, but the shutdown/setup handler itself was not tested. The custom transaction artifact had not yet been installed or enabled at that stage.

The same NVIDIA 610.88 driver, VBIOS and inference-library hashes were retained. Before the stock test, the reported boot-time 450/140 W limits were explicitly changed to the comparison's 400/110 W. Both NVIDIA devices reported error code 0, and no new relevant NVIDIA/display/WHEA System events were found during the observed boot. HWMonitor was version 1.59 for this session, rather than the earlier 1.58.

The full matrix workload marked PASS after completing all eight precision/size cases in 7.7104 seconds, with the same small correctness hashes and finite large outputs. At 8192 the medians were FP16 57.6394, BF16 58.3065, TF32 29.5474 and plain FP32 21.4832 TFLOPS. These are separate observations, not replacement values for the original matched 5 November pair. At that stage no unlocked measurement existed; the later same-day pair is below.

The intended 25-second direct-NVML observation failed at elapsed 12.4231 seconds on the **3080 Ti**. This was 0.3696 seconds after the benchmark marked itself complete, during post-load observation; it must not be described as a failure in a particular timed matrix case. The captured mask was decimal 36 (`0x24`): software power-cap plus software thermal. Hardware thermal was inactive. The simultaneous snapshot recorded core 57 C, P0, 308.115 W, graphics 2025 MHz and memory 9701 MHz.

The software thermal counter (field 269) was 6,876,581,280 ns both initially and at the event. Hardware thermal counter 270 and legacy thermal field 75 remained zero. Memory field 82 explicitly returned NOT_SUPPORTED (3). Successful API calls retained the field values/statuses; unavailable memory was never treated as 0 C. The raw event flag is distinct from the counter discrepancy. [NVIDIA event-reason definitions](https://docs.nvidia.com/deploy/nvml-api/latest/api/group__nvmlClocksEventReasons.html).

HWMonitor's observed session maxima were 62.7 C core, 60 C memory and 73.9 C hotspot, with Temperature Limit maximum still 0. These maxima include the preceding desktop period and are not exact instantaneous readings at the flag. NVML sampled a target maximum of 66 C. Discrete sensor polling and differing telemetry semantics remain relevant uncertainties; neither a low observed temperature nor an unchanged counter establishes that an active flag is harmless.

The supervisor stopped further testing and both owned children exited. **Overall supervised verdict: FAIL**, despite the matrix subtest's eight completed cases. No LLM load or unlocked retry followed in that stock powered session. The A4000 was restored to its ordinary 140 W limit, with the 3080 Ti remaining at 400 W. Both devices remained healthy and the thermal flags were subsequently inactive. The USB remained a non-TX probe with no activation/enrollment markers at that stage; no automatic unlock was armed then.

This establishes that the raw software thermal flag can occur in a boot where the unlock was not applied. It does not establish a driver bug, hardware defect, overheating diagnosis or daily-use acceptance. The exact rounds and event are in [2026-11-06-stock-control.json](data/2026-11-06-stock-control.json). The owner later explicitly requested enabling the transaction controller and another test. The failed stock capture remains part of the interpretation of that later run.

## 6 November unlocked follow-up

The owner subsequently cold-booted the local custom automatic controller. Its actual log recorded the target transaction, DMA release, native display recovery, SS0 `0x88888888`, SS1 `0x8`, feature readout `0`, PLM `0xffffffff`, WPR2 high `0` and D0, followed by the controller's exact final-state verification and Windows StartImage at 19.870 seconds from EFI entry. This is a real transaction, unlike the earlier probe; the elapsed value is not desktop-ready boot time. The custom build was based on v2.1.10 with a separate control layer, not a new claim about the unmodified release asset.

The new log was archived before testing. Automatic repeat activation was temporarily disabled, without undoing the current unlock. Reported boot-time limits again needed adjustment from 450/140 W to the matched 400/110 W. Both GPUs reported device error code 0 and no new relevant NVIDIA/Display/WHEA events were found. No physical-memory diagnostic driver or live register repair was installed.

### Complete matrix comparison

The following same-day numeric pair used the identical bounded matrix script, runtime, seed, small correctness controls, dimensions and rounds:

| Precision, 8192 x 8192 | Stock TFLOPS | Unlocked TFLOPS | Ratio | Gain |
| --- | ---: | ---: | ---: | ---: |
| FP16, FP32 accumulation | 57.6394 | 101.3288 | 1.758x | +75.8% |
| BF16, FP32 accumulation | 58.3065 | 104.9068 | 1.799x | +79.9% |
| TF32 | 29.5474 | 54.2883 | 1.837x | +83.7% |
| Plain FP32 control | 21.4832 | 21.4972 | 1.00065x | +0.1% |

**The stock side's matrix cases completed, but its overall supervision failed on the post-load thermal flag described above.** The unlocked side completed all eight cases and its full 25-second observation passed. Thus the ratios describe completed arithmetic measurements, not two fully accepted stability baselines. Plain FP32 was essentially unchanged; absolute values from the separate initial 5 November run must not be mixed into this pair.

The unlocked direct-NVML observation requested 10 ms polling and counted 2,421 masks per card. Target masks were 1,599 idle, 292 unlimited and 530 power-cap, with zero sampled software/hardware thermal events. Software/hardware thermal counters had zero delta on both cards. Sampled target core maximum was 65 C; secondary core stayed at 36 C during that matrix observation. All small correctness hashes matched the stock control, and large outputs were finite; a full large-matrix numerical reference was not computed.

### Bounded LLM comparison

The exact existing llama.cpp b12032 profile then ran two synthetic cases, two warmups and three measured requests per case. The harness completed all ten requests in 69.704 seconds and stopped its owned server. This was not a ten-minute sustained check. Request settings, context, split, KV types, speculation, executable hash, fixture hash and model path/size/mtime matched the **5 November stock reference**. Full model-weight hashing was not available, as noted earlier.

Long case: 4,876 prompt tokens, 64 generated tokens, seed 42, temperature 0, EOS ignored, prompt caching disabled and `cache_n=0` verified. The measured output hashes matched within each case and against the stock reference.

| Long-case median | Stock reference, 5 November | Unlocked, 6 November | Observed change |
| --- | ---: | ---: | ---: |
| Prompt processing, tokens/s | 457.7052 | 568.8641 | +24.3% |
| Generation, tokens/s | 64.9963 | 75.5140 | +16.2% |
| Request time, seconds | 11.6261 | 9.4194 | 19.0% lower |

The reference is from another day/boot; no fresh stock LLM run followed the failed 6 November stock matrix observation. These matched-profile measurements indicate an improvement in this synthetic setup, but do not isolate every environmental effect or establish the same gain for other models or unseen OpenCode work. Repeated prompts and adaptive MTP/ngram speculation also limit generalization. The short-case results are retained in the JSON rather than presented as headline unseen-code throughput.

The LLM harness used its existing approximately one-second telemetry checks, **not** the matrix observer's 100 Hz raw sampling. Its sampled software/hardware thermal flags stayed inactive and both reported counter deltas were zero. Faster brief events could be missed. HWMonitor 1.59 remained visible; final session maxima across desktop/matrix/LLM use were 63.9 C core, 66 C memory and 74.9 C hotspot, Temperature Limit maximum 0. Its reported session power peak was 405.22 W despite the checked 400 W limit, so an absolute instantaneous power ceiling is not claimed.

HWMonitor's PEX recovery counter had session minimum 64, was 79 before the matrix run and 80 after the tests. Other displayed fatal/nonfatal/CRC/replay error counters were zero. This remains a recorded uncertainty; no hardware-fault or harmlessness diagnosis is inferred. Both GPUs remained device-code 0 and no new relevant NVIDIA/display/WHEA events were observed.

### Current operating state and remaining work

All owned matrix/observer/LLM processes exited. The secondary was restored to 140 W with verified read-back; the target remained at 400 W. Comparison measurements above were at 400/110 W, not the final 400/140 W state. At the owner's request, the saved automatic USB enable file was restored after the bounded passes; no failed or pending latch remained. **Automatic activation is currently enabled on the USB, and the current unlock has not been rolled back.**

This is bounded workload success, not daily hardware acceptance. The earlier stock thermal event remains unexplained. There has been no sustained ten-minute run, no new cold-power rollback after this custom attempt, no validated sleep/resume behavior and no real exercise of the custom error-shutdown/next-power-on setup handler. EFI also cannot detect later Windows workload faults. A future boot's power limits still need checking; the earlier unexplained 450 W boot setting was not fixed here.

Exact stock/unlocked rounds, LLM reference/current request timings, hashes and monitoring summaries are in [2026-11-06-unlocked-follow-up.json](data/2026-11-06-unlocked-follow-up.json). Hardware UUIDs, ROM fingerprint, USB serial, private paths and raw firmware logs are excluded. The local custom controller's source/binary is not distributed in this result update.
