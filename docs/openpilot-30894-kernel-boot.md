# Research Package — openpilot #30894: "Reduce kernel boot time to <1s on comma 3X"

Team F3 groundwork package. **Hardware-blocked** (requires a physical comma 3X / tici device for final measurement and validation). This package maps the terrain, prior art, concrete config/DT changes, expected savings, measurement methodology, and claim strategy. No bounty completion is claimed.

## 1. Bounty requirements (verbatim-ish)

- Issue: https://github.com/commaai/openpilot/issues/30894 (adeebshihadeh, 2024-01-03, labels: `bounty`, `comma three`, milestone "project vamOS")
- Baseline: `systemd-analyze` → `3.650s (kernel) + 7.343s (userspace) = 10.994s`
- Target: kernel portion **<1s**, **self-contained** — "can't just defer stuff to later and cost boot time at later stages"; deferring something non-essential to driving (e.g. WiFi) is explicitly OK (adeeb, issue comment 1925886819).
- "can turn off stuff openpilot doesn't use (NFC, bluetooth, etc.)" — and indeed BT was already disabled (see §3).
- Built via https://github.com/commaai/agnos-builder; kernel submodule = https://github.com/commaai/agnos-kernel-sdm845 (Linux 4.9.103 Qualcomm Android fork, arm64).

## 2. Build-system facts (where changes land)

- `agnos-builder/build_kernel.sh`:
  - `DEFCONFIG=tici_defconfig` → `commaai/agnos-kernel-sdm845: arch/arm64/configs/tici_defconfig`
  - Builds `Image-dtb` (appended DTB), then `mkbootimg` with **no initramfs** (`--ramdisk /dev/null`) — kernel goes straight to mounting rootfs (ext4 on UFS; `build_system.sh` uses `mkfs.ext4`). Good: no initramfs unpack cost.
  - Kernel cmdline already contains: `quiet loglevel=3 console=ttyMSM0,115200n8 earlycon=msm_geni_serial,0xA84000 ... isolcpus=6,7 ... lpm_levels.sleep_disabled=1`. Console silencing already done (adeeb: "We already use `quiet` to get the initial speedup of like 11s → 3s").
  - boot.img signed with `vble-qti.key`; flash via `flash_kernel.sh` (QDL mode).
- Device tree: `agnos-kernel-sdm845/arch/arm64/boot/dts/qcom/comma_tizi.dts` (comma 3X), `comma_mici.dts` (comma four), shared `comma_common.dtsi`, including `sdm845.dtsi` (128 KB!), `sdm845-bus.dtsi`, `sdm845-pinctrl.dtsi`, `sdm845-coresight.dtsi`, PMIC files, panel files. There is already an `unused/` directory and `find_used_dtsi.py` helper — comma has begun DT hygiene.
- Current defconfig (HEAD, post-PR#94/#117): kernel built **uncompressed** (`CONFIG_BUILD_ARM64_UNCOMPRESSED_KERNEL=y`); PCI, CAN, BT, NVMe, SDHCI already `=n`.

## 3. Prior art (measured, from comma themselves)

| Change | Result | Source |
|---|---|---|
| `quiet` on cmdline | ~11s → ~3s kernel | issue comment 1925886819 |
| PR #94 "slim down tici_defconfig for boot time speedups" (Sep 2025): disabled PCI/PCI_MSM, CAN, BT/BTFM_SLIM, WIL6210, NVMe, SDHCI(-MSM), L2TP, MSM_11AD, QPNP_COINCELL, MSM_EXT_DISPLAY, QPNP_FG_GEN3 (later re-added for SOM pwr measurements), MSM_BCL_PERIPHERAL_CTL, KERNEL_FAN | **3.5s → 2.8s kernel** (`systemd-analyze` on device) | https://github.com/commaai/agnos-kernel-sdm845/pull/94 |
| PR #117 "Build uncompressed tici kernel" (May 2026): gzip → uncompressed `Image-dtb` + header fix | **saves ~280 ms** | https://github.com/commaai/agnos-kernel-sdm845/pull/117 |
| adeeb on PR #94: "Doesn't seem like there's much more to save here by just turning off options in the defconfig." | → remaining ~1.8s needs driver/code/DT-level work, not just Kconfig | PR #94 comment |
| andiradulescu plan + `boot.svg` (bootgraph.pl from initcall_debug dmesg) | groundwork posted | agnos-builder#110 comment 1925239170, issue comment 1925854306 |
| Boot-stage tooling: `analyze-boot-time.py` (stage breakdown PON/XBL/ABL/kernel/weston) now lives in **commaai/vamOS** at `userspace/root/usr/comma/tests/analyze-boot-time.py` (removed from agnos-builder at HEAD). It targets vamOS; its stage-split applicability to current AGNOS must be re-confirmed on device. | measurement infra | https://github.com/commaai/agnos-builder/issues/110, https://github.com/commaai/vamOS |

**Implication for claim: the easy ~0.9s of defconfig work is already upstream. The remaining path to <1s (from ~2.5–2.8s) must come from driver init times, probe delays, deferred-probe churn, and DT trim — with an initcall profile to prove each change.**

## 4. Likely remaining boot-time sinks (from defconfig/DTS inspection)

Kernel 4.9 Android-Qualcomm kernels are notorious for long synchronous probes. Ranked suspects:

1. **Synchronous driver probes with msleep/mdelay loops** — display panel (SDE/DSI panel prepare/on sequences), touchscreen (`CONFIG_TOUCHSCREEN_*` × 4 drivers compiled in: synaptics dsx v26 + legacy, hynitron, focaltech/samsung — only one device exists per SKU), camera (`CONFIG_SPECTRA_CAMERA=y`, needed by openpilot camerad — trim its DT, not the driver), audio ASoC machine, USB DWC3 dual-role + QPNP USB-PD, UFS init.
2. **Deferred-probe re-probing churn** — Qualcomm drivers (regulators/rpmh, clocks, SMMU/ARM_SMMU with `CONFIG_IOMMU_DEBUG=y`, GDSC) notoriously bounce through `deferred_probe_work`. Each round re-runs probes. Fixing ordering or pruning DT nodes removes both probe and deferral time.
3. **Bloated DT** — `sdm845.dtsi` is 128 KB and describes MTP/devboard hardware (multiple panels, camera MTP nodes `sdm845-camera-sensor-mtp.dtsi`, coresight topology 47 KB, PCIe, bus scaling). Nodes with `status="ok"` whose drivers are compiled in still get probed. DT unflatten + probe of unused nodes is pure cost.
4. **Debug/observability subsystems compiled in**: `CONFIG_CORESIGHT_*` (~12 drivers), `CONFIG_EDAC_KRYO3XX_ARM64`, `CONFIG_FTRACE` + `IRQSOFF_TRACER`/`PREEMPT_TRACER`/`FUNCTION_GRAPH_TRACER`, `CONFIG_DYNAMIC_DEBUG`, `CONFIG_IKCONFIG`, `CONFIG_KALLSYMS_ALL`, `CONFIG_DEBUG_INFO` (image size → load/copy time from UFS + ABL load), `CONFIG_MEMTEST`, `CONFIG_LKDTM`, `CONFIG_FAULT_INJECTION`, `CONFIG_PSTORE_FTRACE`, `CONFIG_QCOM_RTB`, `CONFIG_TRACER_PKT`, `CONFIG_IPC_LOGGING`, `CONFIG_MSM_BOOT_STATS`, `CONFIG_QTI_RPM_STATS_LOG`, `CONFIG_QCOM_DCC_V2`, `CONFIG_IOMMU_DEBUG/TRACKING/TESTS`, `CONFIG_ARM64_PTDUMP`, `CONFIG_PAGE_OWNER_ENABLE_DEFAULT=y` (page owner is *enabled at boot* — real CPU cost during early alloc), `CONFIG_SLUB_DEBUG`.
5. **Unused but enabled hardware/protocol stacks**: `CONFIG_NFC_NQ=y` (NFC — explicitly allowed to kill), `CONFIG_DVB_CORE`/`DVB_MPQ=m`/`TSPP` + `CONFIG_MEDIA_TUNER*` autoselect (digital TV!), `CONFIG_SND_USB_AUDIO` + `SND_USB_AUDIO_QMI`, `CONFIG_PPP*` suite (PPPoE/L2TP/PPTP…), `CONFIG_BRIDGE*`/`ebtables`, `CONFIG_NFS_FS`, `CONFIG_ECRYPT_FS`, `CONFIG_QUOTA`, `CONFIG_JOYSTICK_XPAD`, `CONFIG_USB_ISP1760` (random host controller), `CONFIG_FB_VIRTUAL`, `CONFIG_LOGO`, `CONFIG_HDMI`, `CONFIG_HW_RANDOM_CAVIUM`, `CONFIG_COMMON_CLK_XGENE`, `CONFIG_PHY_XGENE`, `CONFIG_POWER_RESET_XGENE` (ARM server SoC leftovers), `CONFIG_ESOC*` external-modem stack (3X modem is USB Quectel — verify), `CONFIG_UIO_MSM_SHAREDMEM`, audit (`CONFIG_AUDIT`), SELinux **and** SMACK both enabled (`CONFIG_SECURITY_SMACK=y` while default is selinux).
6. **Module signing**: `CONFIG_MODULE_SIG_ALL=y` with SHA512 — sign cost at build; verify cost at insmod (userspace, but trivial). Low priority.
7. **Randomize base (KASLR)**: `CONFIG_RANDOMIZE_BASE=y` — small relocation cost; `nokaslr` could shave a few ms but probably not worth the security regression for comma.

## 5. Optimization techniques applicable to SDM845 ARM64 (with concrete symbols)

### Tier A — pure removal, zero functional risk for openpilot (est. combined 300–700 ms; needs initcall proof)
- `CONFIG_NFC_NQ=n` (NFC core `# CONFIG_NFC is not set` already at HEAD; only this NXP stub remains — explicitly OK per issue)
- `CONFIG_DVB_CORE=n`, `CONFIG_DVB_MPQ=n`, `CONFIG_TSPP=n`, `CONFIG_MEDIA_DIGITAL_TV_SUPPORT=n`, `CONFIG_MEDIA_TUNER*=n`
- `CONFIG_SND_USB_AUDIO=n`, `CONFIG_SND_USB_AUDIO_QMI=n`
- `CONFIG_PPP*=n` (keep `CONFIG_TUN` if userspace VPNs use it — check openpilot)
- `CONFIG_BRIDGE=n` + ebtables, `CONFIG_NFS_FS=n`, `CONFIG_ECRYPT_FS=n`, `CONFIG_QUOTA=n`, `CONFIG_BLK_DEV_RAM=n` (16×8MB ramdisks!), `CONFIG_ZRAM`? (check userspace), `CONFIG_CRAMFS` (keep — updater uses), `CONFIG_USB_ISP1760=n`, `CONFIG_FB_VIRTUAL=n`, `CONFIG_LOGO=n`, `CONFIG_JOYSTICK_XPAD=n` + joystick/HID extras not used (`HID_MAGICMOUSE` etc. — keep `USB_HID` generic for keyboards during dev), xgene/cavium leftovers, `CONFIG_SECURITY_SMACK=n`, `CONFIG_AUDIT=n` (verify openpilot doesn't read audit), `CONFIG_BLK_DEV_LOOP` (check — overlayfs/updater may use loop), `CONFIG_NL80211_TESTMODE`/`CFG80211_CERTIFICATION_ONUS` minor.

### Tier B — debug/observability trim (CPU + image size; est. 100–400 ms)
- `CONFIG_PAGE_OWNER_ENABLE_DEFAULT=n` (currently **y** — enabled by default!), `CONFIG_SLUB_DEBUG=n`, `CONFIG_DYNAMIC_DEBUG=n`, `CONFIG_DEBUG_INFO=n` (or REDUCED; shrinks Image → faster UFS read + ABL load), `CONFIG_FTRACE=n` + all tracers, `CONFIG_CORESIGHT*=n` (+ delete coresight DT include), `CONFIG_EDAC=n`, `CONFIG_IKCONFIG=n`, `CONFIG_KALLSYMS_ALL=n`, `CONFIG_MEMTEST=n`, `CONFIG_LKDTM=n`, `CONFIG_FAULT_INJECTION=n`, `CONFIG_IOMMU_DEBUG/TRACKING/TESTS=n`, `CONFIG_ARM64_PTDUMP=n`, `CONFIG_MSM_BOOT_STATS`/`IPC_LOGGING`/`QCOM_RTB`/`TRACER_PKT`/`QTI_RPM_STATS_LOG`/`QCOM_DCC_V2=n` (dev-only Qualcomm telemetry), `CONFIG_MMC_PERF_PROFILING`, `CONFIG_SCSI_UFSHCD_CMD_LOGGING=n`.
- Caveat: comma devs use some of these for debugging AGNOS; propose behind a `tici_debug_defconfig` or accept pushback per-symbol.

### Tier C — driver/probe engineering (the big remaining chunk; est. 500 ms–1.5 s, requires device)
- Rebuild dmesg with `initcall_debug loglevel=8 ignore_loglevel` on cmdline (temporarily replacing `quiet`), run `scripts/bootgraph.pl` (kernel tree) → identify top initcalls. Prioritize anything >20 ms.
- Kill fixed `msleep()`/`mdelay()` in: panel power sequences (only keep the *one* panel in use), touchscreen drivers not matching the installed panel (tici uses one of synaptics/hynitron/focaltech per SKU — compile in only the right ones or probe-guard early), USB-PD/QPNP, audio codec, UFS (already `SCSI_SCAN_ASYNC=y`).
- Reduce deferred-probe storms: mark unused DT nodes `status="disabled"` (extra panels in `comma_tizi.dts`, camera MTP sensors, second display, HDMI, coresight, unused QUPv3 ports, unused regulators). Each disabled node avoids a probe attempt + deferral round.
- DT size: strip `sdm845.dtsi` includes not used by comma (PCIe already off — drop `sdm845-pcie.dtsi`; `sdm845-coresight.dtsi`; camera MTP). Smaller FDT = faster unflatten and fewer platform devices created.
- Compile late-needed things as modules where userspace already loads them late (WiFi CLD was moved built-in in commit bf28cdf3 "Build wifi driver with main kernel" — check whether that added boot cost; per adeeb, deferring WiFi is acceptable).
- CPU at max freq early: default governor is already `performance` (`CONFIG_CPU_FREQ_DEFAULT_GOV_PERFORMANCE=y`); `lpm_levels.sleep_disabled=1` already set. Optionally pin `kernel/*` threads — minor.
- `lpj=<value>` cmdline to skip delay-loop calibration (small, ~10s of ms on 4.9).
- Module load: reduce `CONFIG_MODVERSIONS`/SHA512 sig hash to SHA256 (marginal).

### Tier D — structural (higher risk/effort)
- Move to a newer/mainline kernel (comma roadmap: openpilot#32386) — out of scope but changes the calculus; don't duplicate if vamOS migrates.
- Async probe allowlist via `driver_async_probe=` for slow-but-independent drivers (UFS already async via SCSI_SCAN_ASYNC).
- Shrink `Image-dtb` further (B Tier + `CC_OPTIMIZE_FOR_SIZE` test) — ABL loads kernel from UFS; uncompressed kernel is ~50+ MB with debug info; DEBUG_INFO off helps ABL stage too (out of kernel-time scope but free).

## 6. What openpilot actually uses — do NOT disable (risk register)

Derived from defconfig history, openpilot source usage, and reverted commits:

| Subsystem | Status | Evidence |
|---|---|---|
| UFS storage (`SCSI_UFSHCD`, `UFS_QCOM`, ICE) | REQUIRED — rootfs | obvious |
| Display SDE/DSI + KGSL/Adreno (`DRM_MSM`, `QCOM_KGSL`, panel dtsi) | REQUIRED — UI | screen bringup commits |
| Touch (one of synaptics dsx / hynitron / fts/samsung) | REQUIRED — UI; keep exactly the in-use driver(s) | "Hynitron touch driver #52", synaptics fw flasher #21 |
| Camera Spectra (`CONFIG_SPECTRA_CAMERA`, CAMCC) | REQUIRED — camerad drives OX03C10 via V4L2 | tici design |
| Video (`MSM_VIDC_V4L2`) | used by encoder (loggerd) | openpilot uses HW encode |
| Audio (`SND_SOC_MACHINE_SDM845`, generic codec) | REQUIRED — sounds/alerts | "Generic codec #63" |
| IMU (`INV_MPU_IIO_ICM42600`, SPI GENI) | REQUIRED — sensord/locationd | defconfig tail |
| I2C/SPI QCOM GENI, pinctrl, SPMI PMIC, QPNP charger/FG, RTC_QPNP, thermal/TSENS, INA2XX, RPR0521 light sensor | REQUIRED — power, fan control, thermal, sensord | defconfig, QPNP_FG_GEN3 re-added (commit 0d9972cc) |
| USB DWC3 + gadget NCM (`USB_CONFIGFS_NCM`) | REQUIRED — tethering/EON debugging | tici networking |
| WiFi QCACLD (`QCA_CLD_WLAN`, ICNSS, CNSS_UTILS) | REQUIRED but deferrable | adeeb: "something non-essential to driving like WiFi is fine" to defer |
| Modem/QMI/RMNET/IPA (`QMI_INTERFACE`, `RMNET_DATA`, `IPA3`, `USB_NET_QMI_WWAN`, `ESOC`?) | REQUIRED — LTE (quectel). Verify ESOC vs USB path before cutting ESOC. | defconfig |
| SELinux (permissive) | keep; cmdline sets permissive | cmdline |
| pstore, minidump, watchdog v2 | keep — comma relies on crash telemetry | commits |
| NFS | dev-only; safe to drop | commit 6595bbfa |

## 7. Measurement methodology (what comma will accept)

1. **Primary metric**: `systemd-analyze` kernel time on a comma 3X, on stock latest AGNOS vs patched boot.img — this is the bounty definition. Report ≥5 cold boots each, median + min/max.
2. **Stage-split tooling**: `analyze-boot-time.py` — now in **commaai/vamOS** (`userspace/root/usr/comma/tests/analyze-boot-time.py`); it no longer exists in agnos-builder at HEAD (404). It was written for vamOS stage timestamps (PON/XBL/ABL/kernel/weston); confirm it (or an AGNOS equivalent) still produces a valid stage split on current AGNOS before relying on it. Its purpose — proving no cost shifted to other stages (bounty constraint) — stands; show ABL/userspace numbers unchanged.
3. **Profiling**: cmdline `initcall_debug ignore_loglevel printk.time=1` → dmesg → `scripts/bootgraph.pl` (boot.svg) + `scripts/bootgraph.pl`/`analyze_suspend`-style sorting of initcalls; andiradulescu already published a boot.svg (agnos-builder#110) — regenerate at HEAD.
4. **Functional validation**: full openpilot onroad test (drive or replay), `lsusb`/wifi/LTE/camera/thermal checks; comma's AGNOS release checklist in agnos-builder README; CI build of kernel+system images.
5. Flash process: `./build_kernel.sh` → `./flash_kernel.sh` (QDL).

## 8. Prioritized savings estimate (baseline ~2.5s kernel, post-PR#94/#117 — i.e. 2.8s minus ~280ms uncompressed-kernel win; note: ~2.5s is an extrapolation, not a measured HEAD number)

**Calibration caveat:** all Tier A–C savings ranges below are uncalibrated engineering guesses until a HEAD `initcall_debug` / boot.svg profile exists on real hardware; treat them as prioritization hints, not commitments.

| # | Change | Est. saving | Risk | Confidence |
|---|---|---|---|---|
| 1 | Tier A unused subsystems (NFC_NQ, DVB/tuners, USB audio, PPP, BRIDGE, NFS, ECRYPT, ramdisk blk, ISP1760, joystick/extra HID, xgene/cavium, SMACK, AUDIT) | 0.2–0.5 s | low | medium (initcall-dependent) |
| 2 | Tier B debug trim (PAGE_OWNER default off, DEBUG_INFO off, FTRACE off, CORESIGHT off, IOMMU debug, Qualcomm telemetry) | 0.1–0.4 s | low-med (dev pushback) | medium-high |
| 3 | DT trim: disable unused panels/sensors/PCIe/coresight nodes; strip MTP includes | 0.2–0.6 s | med (must match all shipped SKUs) | medium |
| 4 | Fix top initcall delays (panel/touch/USB-PD/audio probe sleeps, deferred-probe ordering) | 0.5–1.2 s | med-high (needs device) | high that this is where the remaining time is |
| 5 | Kernel image size reduction → faster UFS read/ABL load (also helps XBL/ABL stage, doesn't count but free) | n/a for metric | low | high |
| 6 | WiFi defer to module load post-boot (if #77 regression) | 0.1–0.3 s | allowed by adeeb | low-med |

Total plausible: ~1.3–2.5 s against a ~2.5s baseline → barely reaches <1s, and only if #4 (initcall/delay engineering) lands on real hardware. This is consistent with adeeb's assessment that defconfig-only work is exhausted.

## 9. Claim strategy (when hardware is available)

1. Reproduce baseline on latest AGNOS: `systemd-analyze` + stage-split tooling, 5 cold boots. (All savings ranges in §5/§8 are uncalibrated guesses until a HEAD `initcall_debug`/boot.svg profile exists — producing that profile is step 1 of real work, and andiradulescu's 2024 boot.svg is stale.)
2. `initcall_debug` boot → bootgraph → publish the profile (fill the gap andiradulescu left).
3. Land changes as small, separately-measured PRs to `commaai/agnos-kernel-sdm845` (defconfig/DTS) and `commaai/agnos-builder` (cmdline) — mirrors how #94/#117 were accepted. Each PR: before/after systemd-analyze numbers + bootgraph delta.
4. Prove self-containment: userspace/ABL times unchanged; no service ordering changes; onroad test passes; all Tier-6 hardware functional.
5. Final claim comment on #30894 with: PR links, systemd-analyze <1s output, boot graph, test evidence. (comma bounty terms: first *merged* PR meeting the criteria wins.)

## 10. Sources

- Issue & comments: https://github.com/commaai/openpilot/issues/30894
- agnos-builder: https://github.com/commaai/agnos-builder (README, `build_kernel.sh`, issue #110 + comments)
- Kernel: https://github.com/commaai/agnos-kernel-sdm845 (`arch/arm64/configs/tici_defconfig`, `arch/arm64/boot/dts/qcom/comma_tizi.dts` et al.)
- Prior-art PRs: https://github.com/commaai/agnos-kernel-sdm845/pull/94 (3.5→2.8s), https://github.com/commaai/agnos-kernel-sdm845/pull/117 (~280ms), commits 77d5e028, b330d9e7, 0d9972cc, bf28cdf3
- Community methodology: andiradulescu plan + boot.svg (issue comment 1925854306; agnos-builder#110 comment 1925239170)
- Techniques: Bootlin boot-time training slides (bootlin.com/doc/training/boot-time), elinux.org/Boot_Time, elinux.org/Initcall_Debug, Android boot-time-opt guide (source.android.com/docs/core/architecture/kernel/boot-time-opt) — all linked from the issue thread itself.

*Prepared read-only by Team F3. All repo facts verified at agnos-builder HEAD 8f7207a / kernel HEAD 21370d6 (as of research date). No bounty completion claimed.*
