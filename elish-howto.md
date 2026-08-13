# elish HOWTO: build & flash a working kernel with KernelSU (ReSukiSU) + SuSFS

This document explains how this fork produces a kernel for **Xiaomi Pad 5 Pro (elish, sm8250)** that:

- boots correctly on **LineageOS 23.2 (Android 16)** (the official APTKernel releases hang on elish),
- includes a **kernel-level root** (ReSukiSU, a KernelSU fork for non-GKI kernels) plus **SuSFS** root hiding.

> **Disclaimer:** this is **personal, unofficial documentation — treat it as one person's experience, verify
> against your own device and the upstream docs.** It's not guaranteed to apply to every elish / every ROM / every
> LOS build, and flashing correctly is your own responsibility. Use at your own risk.

---

## 1. Why this fork differs from upstream

| Commit | Change | Why |
|---|---|---|
| `5fb7151` | `elish_defconfig`: disable `CONFIG_SHADOW_CALL_STACK` | **Fixes the boot hang on elish / LOS 23.2** (stuck on the MI logo). This is the standard fix the upstream maintainer gives for the sm8250 bootloops (see upstream issue #30). |
| `5a51b9a` | `build_kernel.sh`: enable `CONFIG_KSU_DEBUG` for AOSP builds | Diagnostic only: makes `allow_shell=true` so adbd (uid 2000) can `su` directly. Useful to break the chicken-and-egg of "no ksud -> no su -> can't install ksud". |
| `8ea0f3e` | `build_kernel.sh`: drop `CONFIG_KSU_DEBUG` again | Final **release** kernel: no unconditional adbd root. Keeps SCS disabled and ReSukiSU + SuSFS enabled. |

> The current recommended production build is commit `8ea0f3e`.

The KernelSU sources are injected at build time by the upstream ReSukiSU `setup.sh`; see `build_kernel.sh` (it runs
`curl .../ReSukiSU/kernel/setup.sh | bash` before compiling). SUSFS is compiled in via Kconfig
(`CONFIG_KSU_SUSFS_*`), so no extra userspace module is required.

---

## 2. Build

Trigger the GitHub Actions workflow on the `android16-aptusitu-new` branch:

```bash
gh workflow run build.yml \
  -R <this-fork> \
  -f device=elish -f ksu=true -f target_os=aosp
```

- Inputs: `device=elish`, `ksu=true` (enables ReKernelSU), `target_os=aosp`.
- Toolchain: zyc-clang 16 + ccache; roughly 15–60 min.
- Artifact: an AnyKernel3 zip (`APTKernel_*_anykernel3_*.zip`).
- For a debug build (adbd shell becomes root), re-add `-e KSU_DEBUG` in the AOSP config block of `build_kernel.sh`. **Do not ship that to a daily driver.**

You can also build locally: `./build_kernel.sh elish ksu aosp`.

---

## 3. Flash

The kernel ships as **AnyKernel3** and must be flashed from **recovery via ADB sideload** — do not `fastboot flash` the bare `Image`.

1. Boot to **bootloader fastboot** and flash the stock boot images from your current LOS zip:
   ```bash
   fastboot getvar current-slot   # MUST return a slot; otherwise you are in fastbootd, not real fastboot
   fastboot flash boot          boot.img
   fastboot flash vendor_boot   vendor_boot.img
   ```
   > If `fastboot flash` seems to hang on `Sending 'boot'` and every `getvar` fails, you are in **fastbootd** (user-space fastboot from recovery). Leave it with `fastboot reboot-bootloader`.
2. Reboot to recovery → **Apply update → Apply from ADB**, then:
   ```bash
   adb sideload APTKernel_*_anykernel3_8ea0f3e2-nodebug.zip
   ```
   Recovery will warn the archive is unsigned — choose **Install anyway** (normal for AnyKernel3).
3. Reboot into the system.

---

## 4. Verify

```bash
adb shell id             # expect uid=2000(shell)  -> release build, KSU_DEBUG off
adb shell 'su -c id'     # expect uid=0(root), context=u:r:ksu:s0  -> root works once granted
```

SUSFS enables hiding (mount / path / kstat / uname / cmdline / open-redirect). With root detections that run
unprivileged (i.e. no superuser grant), the device reports **not rooted**; only apps you explicitly grant in the
manager see root.

---

## 5. Manager (SukiSU-Ultra) — pick the matching build

The kernel's `KSU_APP_PROFILE_VER` is **4**. The manager must send the same profile protocol version, otherwise the
manager cannot grant superuser to apps ("Failed to set app profile", kernel log: `Unsupported profile version: N`).

| Manager build | `KSU_APP_PROFILE_VER` | Works with this kernel? |
|---|---|---|
| release v4.1.2 (40545) | 2 | no |
| release v4.1.3 (40796) | 3 | no |
| **main-branch CI build v4.1.3 (40856)** | **4** | **yes** |

Grab the `manager` CI artifact from SukiSU-Ultra's Actions, or build from its `main` branch yourself. The stock
Play-store / release APK is protocol-incompatible with this kernel's ReKernel 4.2-rc profile.

> If you ever see the manager prompting you to "select GKI image / patch boot", that does **not** mean you should go
> patch a GKI image — it just means the manager did not detect the (built-in, non-LKM) driver. Align the protocol
> version instead.

---

## 6. Troubleshooting pointers

- **Stuck on MI logo**: SCS is the usual culprit on elish/LOS23 — verify `# CONFIG_SHADOW_CALL_STACK is not set` in `elish_defconfig`.
- **`su: inaccessible or not found`**: `/system/bin/su` is rewritten to `/data/adb/ksud`; if ksud is absent, `su` is invisible. Let the manager install ksud, or use a `KSU_DEBUG` build once to break the deadlock.
- **Manager shows mismatch / can't grant**: compare `dmesg | grep "Unsupported profile version"` against the table in §5.