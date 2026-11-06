# Reproducing the investigation

This is the order of checks used for one experimental system, with failures retained. **It is not a validated daily-use recipe or a promise of hardware safety.** Read the [result and limitations](REPORT.md) before preparing media. A later bounded unlocked application check passed, but the thermal-flag explanation and sustained/daily acceptance remain outstanding.

## 1. Establish your own target and recovery path

Record your GPU model, PCI device/subsystem IDs, VBIOS, driver, physical PCI address, power limits and active display connections. Do not copy this experiment's bus addresses, USB drive letter, power limits or UUIDs. GPU numbering can differ between Windows, CUDA and nvidia-smi.

Read-only Windows inventory examples:

```powershell
nvidia-smi --query-gpu=name,uuid,pci.bus_id,driver_version,vbios_version,power.limit,power.default_limit,power.min_limit,power.max_limit --format=csv
Get-CimInstance Win32_VideoController | Select-Object Name,PNPDeviceID,ConfigManagerErrorCode
Get-Disk | Select-Object Number,FriendlyName,BusType,Size,PartitionStyle
Get-Volume | Select-Object DriveLetter,FileSystemLabel,FileSystem,DriveType,Size
```

Back up tuning profiles and verify that overclock/undervolt/startup scripts are not reapplying during the comparison. Check effective limits after every boot. Do not apply this machine's 400/110 W values indiscriminately to another board. Keep all comparison settings identical, within each card's supported range.

Verify power delivery and a real alternate display/recovery path. Know how to select ordinary Windows Boot Manager without the USB. Privately verify any encryption recovery key before changing the boot path; do not put recovery secrets into logs. Check your platform's UEFI/Secure Boot requirements against the pinned project. This report did not require changing security settings and does not instruct you to disable them.

## 2. Pin the actual Windows implementation

The Linux `make install` instructions at the repository root do not install the UEFI program used here.

Start with the exact [v2.1.10 release](https://github.com/physicsdolphin/nv-uefi-unlocker/releases/tag/v2.1.10) and its [tagged source](https://github.com/physicsdolphin/nv-uefi-unlocker/tree/v2.1.10). Record commit `fbdcc5b6e7974864d867f8c878813257b7ab7ca5`, compare release hashes and review the tagged English FAQ and maker before running anything. A later version needs a new review; current `main` is not evidence for the tagged binary.

The runtime filenames, sizes and observed hashes are in [artifact-manifest.json](data/artifact-manifest.json). Obtain/generate inputs through the pinned project's reviewed workflow. Firmware, EFI binaries and model files are intentionally not included in this fork. Never download a ready-made installer from an unrelated mirror on the strength of this report.

For a source checkout with the required Rust/MSVC/UEFI tooling, the pinned Cargo aliases used were:

```powershell
cargo xt
cargo efi
```

`cargo xt` runs host-side workspace tests excluding the EFI loader. `cargo efi` builds the **read-only** loader without `tx`. Verify the alias and actual build flags in that checkout. A different artifact is used for a transaction. Keep read-only and transaction artifacts in separate directories and inspect their mode banners/hashes with the pinned project's checker; its read-only verdict used exit code 2, while a transaction verdict used exit code 0.

The tag's Cargo.lock had four workspace package versions at 2.1.9 while the manifests were 2.1.10. Only those four package versions were aligned for the local read-only build; dependency versions/checksums were unchanged. The exact [lockfile metadata patch](reference/lockfile-metadata-fix.diff) is included for provenance, not an instruction to alter unrelated dependencies. The transaction EFI was the original pinned release asset, not a locally modified transaction build.

Passing host tests does not test your motherboard's UEFI, DMA/reset path or recovery behavior. Removing a payload file from a transaction build does **not** make it read-only.

## 3. Inspect external media before any transaction

Use identified, backed-up external FAT32 media. Re-read its physical device identity, capacity and drive letter immediately before copying. This report supplies no formatter and no internal ESP/BCD/BootOrder installer.

First boot the separately verified read-only artifact through the one-time UEFI boot menu. Preserve its new log, check enumeration and normal Windows handoff, and verify both devices/displays afterwards. Stop if that phase fails.

For the later transaction layout, the eight runtime paths are listed in the manifest. The pinned `tx-select-bdf.txt` belongs in the USB root and contains the **freshly verified target's full BDF**, ASCII, with an optional line ending. Family selection such as `ga102` is not single-GPU selection.

**Important v2.1.10 targeting limitation:** an absent selector defaults to all supported GPUs; malformed or unmatched content behaves differently and stops selection. A forgotten/misplaced selector must never be treated as safe exclusion. The A4000 can also match the supported family. If you require guaranteed exclusion of another card, do not proceed in that configuration based only on this file. This experiment accepted the residual missing-file risk; it did not fix the firmware's default.

Do not create `i_know_what_i_am_doing.txt` or optional diagnostic/workaround markers for this manual procedure. Manual confirmation happens before selection, so a confirmation screen is not proof that the final target set is correct. Verify the actual destination files, mode, selector and hashes, and preserve the prior loader log before another boot.

## 4. Complete a stock baseline first

Run stock without unlock media, using controlled tuning and the same runtime/libraries intended for comparison. Monitor **both** cards with HWMonitor and nvidia-smi. Read memory and hotspot temperatures explicitly rather than assuming the core reading covers them.

The historical matrix test used seed 1729, 4096/8192 sizes, four warmups, three measured rounds of ten GEMMs, reduced-precision accumulation disabled for FP16/BF16, TF32 and plain FP32 controls, a small CPU-reference equality control and large-output finiteness checks. It was bounded; it was not an unlimited stress test.

Our conservative stop criteria were core >=80 C, memory >=90 C, hotspot >=95 C, active software/hardware thermal flags, device/driver errors or required telemetry failures. They were experiment-specific cutoffs, not NVIDIA damage thresholds. Unsupported memory readings remained unsupported and required separate HWMonitor observation. Memory/hotspot supervision was manual where NVML could not provide the data.

A host-library CPU crash needed a process-local cuBLASLt workaround, described in the report. Do not silently change the runtime between phases to obtain a speedup. A failed baseline is a stop, not a reason to try the unlock as a repair.

For an application comparison, use the exact same local model file/tokenizer, engine build, context, KV types, GPU order/split, offload, speculation and sampling parameters. Use public/synthetic prompts, two warmups and at least three measured repetitions; collect prompt processing, generation and request wall time separately. Disable/verify prompt caching and record adaptive speculation effects. Do not publish private chats, keys or model binaries.

## 5. Manual firmware attempt and bounded acceptance

Only a physically present owner with a working recovery path should make the firmware decision. Save work, stop other GPU workloads deliberately and follow the pinned project's cold-start and manual one-time boot instructions. Do not change permanent boot order as part of this checklist.

Black/frozen target output can occur during reset, so alternate output and observable progress matter. Our offline recovery plan treated five minutes with no Windows **and no observable progress** as a suspected hang; continuing USB/output activity meant the timer alone was not a reason to cut power. That timeout was not a hardware guarantee, and forced power removal is not harmless by definition. Have your platform-specific recovery procedure available independently of this chat/report.

After Windows returns, archive the **new** transaction log before another boot. Verify one intended target/attempt, stage results, post-FLR readout, cleanup and Windows handoff. Check device health, displays and new driver/display/WHEA events. Re-check the comparison limits after boot before starting any load.

Then repeat only the bounded, supervised matrix suite. On any stop, preserve the offending GPU/field/readings and stop further unlocked testing in that powered session. Do not jump directly to an LLM or ten-minute workload to see whether it goes away.

Only a clean complete compute/numerical pass justifies a matched application test, then a separately supervised sustained test. The initial attempt stopped early; the later 6 November attempt completed the full matrix supervisor and bounded synthetic LLM check. A sustained run remains outstanding. The archived twelve CPU-only tests do not certify the GPU; consult the actual hardware evidence and its stated limits.

## 6. Demonstrate rollback and restore ordinary settings

Our owner shut down, removed PSU power until the card lost power, removed the unlock USB and booted normal Windows. A complete stock compute/inference comparison returned to baseline. This is the observed recovery; it cannot guarantee that every possible firmware failure is recoverable.

Restore any temporary supported power limit to its previously recorded value after testing. Keep private backups of tuning/startup settings. Do not re-enable saved tuning without checking what it applies. Preserve negative results as well as gains.

## Booting, Windows repair and gaming

The measured unlock is volatile: full card power loss cleared the initial attempt. The external-USB arrangement needs the pre-Windows stage after such a loss, not for each app launch while state survives. Sleep/resume and resets remain unvalidated. The owner later requested and enabled a local custom automatic USB controller; its successful bounded result is documented, but its failure-shutdown/setup path and sustained use are not validated. It is not distributed as an installer here. No internal ESP/BCD/BootOrder installation was made.

The separate Windows `ssdiag --reapply` tool can repair selectors only while the necessary FEAT PLM gate remains open; it cannot open the gate after cold power loss. It uses a physical-memory kernel driver and was **not installed or tested here**. See the [tagged ssdiag documentation](https://github.com/physicsdolphin/nv-uefi-unlocker/blob/v2.1.10/tools/ssdiag/README.md) and [FAQ](https://github.com/physicsdolphin/nv-uefi-unlocker/blob/v2.1.10/docs/faq.en.md) rather than assuming it is an ordinary Windows startup replacement.

The 3080 Ti runs games in its stock state. This compute unlock does not select the GPU for a game, move a display connection, add VRAM or establish a gaming benefit. Investigate app GPU selection and monitor connections separately. A later report that a game used the A4000 was not reproduced during this investigation and no routing fix is claimed.
