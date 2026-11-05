# Historical diagnostic sources

These are sanitized snapshots of the investigation scripts, provided as `.py.txt` **reference text**, not a supported portable tool or installer. They retain historical machine-specific assumptions. They have not been made generally configurable or validated on another system. The latest supervision changes have CPU-only coverage but no subsequent unlocked hardware run.

| File | Purpose and limitations |
| --- | --- |
| `gpu_benchmark.py.txt` | Bounded PyTorch comparison and nvidia-smi guard; `GPU-TARGET` is a placeholder, target model/400 W policy were specific to the tested board. It creates GPU load if deliberately adapted and executed. |
| `observe_nvml.py.txt` | Passive direct-NVML observer; `GPU-TARGET` and `GPU-SECONDARY` are placeholders. No setter, CUDA context or kernel-driver installation. Requires the Windows NVML DLL and the specific API/counter capabilities; unsupported required fields fail closed. It does not stop unrelated workloads. |
| `trace_stock_matrix.py.txt` | Parent process coordinating only its own observer and benchmark children. Environment paths and secondary 110 W policy are experiment-specific. Memory/hotspot supervision remains manual. |
| `test_diagnostic_guards.py.txt` | Twelve CPU-only tests for telemetry/stop/supervision logic. Importing the reference modules does not load NVML or PyTorch; hardware APIs are inside main functions. |
| `llm_benchmark.py.txt` | Historical exact-profile llama.cpp test harness; private paths/profile names were replaced. Its local engine/config are not bundled, so this is methodological reference, not an executable reproduction command. |
| `llm-fixtures.json` | Exact public synthetic prompts and request settings used in the comparison; its byte hash matches `fixtures_sha256` in the inference data. The old note mentions numbered prefixes, but the final harness sent identical per-case prompts without them. |
| `lockfile-metadata-fix.diff` | Exact four-entry workspace-version metadata correction for the pinned derivative's read-only source build. It is not a patch to this fork's Linux implementation. |

Actual UUIDs are replaced consistently, and user paths become `<LOCAL_WORKSPACE>`, `<LOCAL_RUNTIME>`, `<LOCAL_MODEL>`, `<LOCAL_CONFIG>` or `<USERPROFILE>`. These markers deliberately do not identify a usable local device/file. Do not substitute another user's addresses or blindly remove a guard to make a reference run.

To inspect the CPU-only tests, copy only the three named module snapshots into a disposable directory and remove their `.txt` suffixes: `observe_nvml.py`, `trace_stock_matrix.py`, `test_diagnostic_guards.py`. Then run:

```powershell
python -m unittest -v test_diagnostic_guards
```

This tests stop logic with synthetic telemetry and simulated child processes. It does not launch an observer, GPU matrix operation, LLM server, firmware program or administrative helper. The publication was checked this way with 12 tests passing.

The matrix snapshots come from the already tested investigation code; placeholder substitutions change device/path literals, not thresholds, sizes or numerical operations. No post-publication adaptation has been hardware validated. All future hardware testing must follow the [reproduction gates](../REPRODUCING.md), preserve supervision, and verify fresh device identity and power settings.

## Diagnostics that must not be overinterpreted

- `--abort-on-thermal` exits the observer, and the parent can then stop its own child. It is not a hardware cutoff or protection for an unrelated process.
- The parent requests termination at its roughly 50 ms polling cadence plus scheduling latency. A silent observer can take up to the three-second heartbeat threshold to be detected.
- Some telemetry is sampled once per second or on a mask transition; short events can be missed. The API status and field timestamp must remain available in evidence.
- NVML memory status 3 means unavailable, never 0 C. Hotspot is not monitored by these scripts. HWMonitor must be supervised separately where required.
- Core 80 C, memory 90 C and hotspot 95 C are conservative experiment cutoffs, not manufacturer damage specifications.
- Historical counter semantics are inconsistent in the stock observations. Keep counters and raw active masks separate; do not suppress an active thermal flag to obtain a pass.

Private startup/power/USB administrative scripts are deliberately not copied: they identify a particular machine and can change its configuration. No driver, boot-media writer, firmware binary or unattended activation marker is distributed with this reference directory.
