# Q2BOX — `.apk` Oculus → PCVR (Windows) Overlay Layer

A local emulation tool attempting to run **Quest** VR games (Oculus `.apk`) on a Windows PC using **3 core components**:

1. **Android ARM64 Emulation** (QEMU `qemu-system-aarch64` backend or Android Studio SDK) — boots an Android ARM system image and installs the APK.
2. **Oculus Bypass** — patches `smali`/entitlements, spoofs Quest device identity (`ro.product.*`), then decompiles, rebuilds, and re-signs the APK.
3. **Windows OpenXR** — native C++ runtime (`xrNegotiateLoaderRuntimeInterface` compliant with the OpenXR 1.1 spec) + stereo window capture for the **headset**.

---

## ⚠️ Technical Reality Check — Read BEFORE Proceeding

> A Quest game is an **Android ARM64** binary calling `OpenXR Android` (`libopenxr.so`) and the Oculus framework (entitlement, VrService, `ro.product.brand=oculus`…).
> A Windows PC uses **x64** architecture and **Windows OpenXR**. There is no official bridge between these two worlds.

**What this overlay layer actually achieves (currently functional):**

- `questbox patch`: Patches entitlement checks (`OVREntitlementChecker`, etc.) for **offline** side-loading.
- `questbox emu`: Boots an Android ARM64 image inside QEMU (TCG acceleration, no KVM under Windows — **expect ~10-20 FPS**, sufficient for testing lightweight games, not Beat Saber natively).
- `questbox xr install`: Registers the Q2BOX Windows OpenXR runtime (`active_runtime.json` + registry key `HKLM/HKCU\Software\Khronos\OpenXR\1\ActiveRuntime`).
- `questbox capture`: Captures the emulator window (GDI, BMP) — ready to feed the native runtime's swapchain.

**What is NOT magic**: High-profile Quest titles (`Beat Saber`, `Population One`, …) rely on native single-ARM GPU rendering + `OVRManager`. Running them on Windows requires either extremely heavy ARM emulation (unviable at 72 Hz) or a native port. This framework provides a **functional foundation** for lightweight/dev VR games and a **testing base** to extend the native renderer (see `native/`).

---

## 💻 PC System Requirements

The hardware requirements for **Q2BOX** are split into two tiers: the baseline specs required to run the current build tools, asset unpackers, and headless translation workflows, and the recommended specs to expect a **somehow "smooth" gameplay experience** (60–72+ FPS stereo rendering) once the visual pipeline is integrated.

| Component | Minimum Requirements (Build / Headless Preview) | Recommended Requirements (To expect "smooth" gameplay) |
| :--- | :--- | :--- |
| **OS** | Windows 10 (64-bit, v1909 or higher) | Windows 11 (64-bit) |
| **Processor (CPU)** | Intel Core i5-8400 / AMD Ryzen 5 2600 (6 Cores with AVX2 support) | Intel Core i7-12700K / AMD Ryzen 7 5800X or higher (High single-core & multi-thread IPC) |
| **Memory (RAM)** | 16 GB DDR4 | 32 GB DDR4/DDR5 (High speed required for real-time asset streaming) |
| **Graphics (GPU)** | NVIDIA GTX 1060 / AMD RX 580 (6GB VRAM) | NVIDIA RTX 3070 / AMD RX 6700 XT or better (For dual-eye stereo OpenXR presentation) |
| **Storage** | 20 GB free space (SATA SSD / HDD) | 50 GB free space (NVMe M.2 SSD required) |
| **Software Stack** | Python 3.10+, Java JDK 11+, VS2022 Build Tools | Python 3.11+, OpenXR Tools for Windows, Latest GPU Drivers |

---

## Installation

External tool prerequisites (configurable in `config.json` → `tools`):

| Tool | Role |
| --- | --- |
| `platform-tools/adb.exe` | ADB wrapper (install, shell, launcher) |
| JDK/JRE 11+ | Runs `apktool.jar` & `apksigner` |
| `apktool.jar` | Decompiles, patches, and rebuilds smali code |
| `apksigner.bat` (Android build-tools) | Re-signs APK files |
| `qemu-system-aarch64.exe` | Boots Android ARM64 (if using the QEMU backend) |

**1. Configure paths**: Edit `config.json` (or launch interactive setup via `questbox.bat`).  
**2. Place the Android ARM64 image** inside `images/android-arm64/` (`system.img` + `kernel` + `ramdisk.img`) for the `qemu` backend.  
**3. Verify setup**:

```bat
python main.py doctor
```

---

## Usage

```console
rem Run diagnostics
python main.py doctor

rem Analyze an Oculus APK
python main.py analyze Example_VR.apk

rem Bypass: Patch entitlements + rebuild + re-sign
python main.py patch Example_VR.apk            (preview: --dry-run)

rem Boot the Android emulator (QEMU or Studio SDK)
python main.py emu [--backend qemu|studio]

rem All-in-one execution: boot + install + launch
python main.py run Example_VR.apk --dry-run

rem Windows OpenXR Runtime management
python main.py xr status
python main.py xr install      # Activates the Q2BOX runtime
python main.py xr layer        # Registers the API layer

rem Capture the emulator display window
python main.py capture --title QUESTBOX --frames 60
```

Or via `questbox-gui.bat` (idempotent shell shortcut: `questbox run my_game.apk`).

<img width="1131" height="768" alt="{C171A274-72CC-433D-B642-6DB7ABA1422E}" src="https://github.com/user-attachments/assets/2ce02bb8-352b-4680-9c59-6bce23db50f5" />

---

## Codebase Architecture

```
config.json                       Paths & settings (tools, android, oculus, runtime, output)
questbox/
  config.py                       Template + deep merge + path helpers
  log.py                          Console & file logging
  _util.py                        Process execution via subprocess & wait_until helpers
  tools/
    adb.py                        ADB wrapper (devices, install, shell, am start)
    apk_analyze.py                VR stack detection (libopenxr/OVR/Unity...), ARM64 ABI checks
    apk_build.py                  apktool (decode/rebuild) + apksigner (signing)
  emulator/
    manager.py                    Backend selector
    qemu.py                       Boots Android ARM64 in QEMU (tcp:5555 -> adb)
    studio.py                     Android SDK backend (x86 AVD) for non-ARM tests
  oculus/
    analyze.py                    Analysis support tools
    entitlement.py                Scans smali & patches methods (returns true/VAR)
    spoof.py                      Spoofs Quest device properties (ro.product.*) via resetprop
    patch.py                      Full patch workflow orchestration (decode->patch->rebuild)
  openxr/
    runtime.py                    Manifest active_runtime.json + Windows Registry (HKCU/HKLM)
    capture.py                    Window GDI screen capture (capture_frame, save_bmp)
native/openxr_runtime/
    OpenXRRuntime.cpp             Native C++ OpenXR runtime (negotiation + core function skeletons)
    runtime.json                  OpenXR runtime manifest (file + api_version)
    CMakeLists.txt  build.ps1     Native build scripts (VS2022)
```

---

## Building the Native OpenXR Runtime

```console
cd native\openxr_runtime
powershell -ExecutionPolicy Bypass -File build.ps1
```

Then run `python main.py xr install` from the project root.

**Next Integration Steps** (natural progression):
- Inside `OpenXRRuntime.cpp`, structure the `swapchain`: Python display captures generate `CaptureFrame` RGBA buffers; the D3D11/Vulkan implementation needs to `UpdateSubresource`/blit into the swapchain texture and present dual-eye views (`-device` viewport left/right).
- Cleanup `questbox xr layer` registry modifications to only apply on demand.

---

## Limitations & Ethics

- This is a **reverse-engineering** tool designed for developers, research, and educational testing.
- Meta/Oculus actively combats entitlement bypasses; this framework should only be used with APKs you own or have explicit rights to redistribute (e.g., your own dev builds / personal sideloads).
- On Windows hosts, **there is no native ARM GPU acceleration** inside the emulator: TCG-translated execution is inherently slow (~10–20 FPS, suitable for testing logic, not 72 Hz real-time gameplay).

---

## 🚦 Current Project Status & Functional Scope

While the underlying core components of Q2BOX are functional and operating as intended, the software is currently in an early development phase:

* **Core Pipeline:** Functional. The tool successfully unpacks APK packages, parses ARM64 binaries, translates `.so` libraries to `.dll`, and initializes the headless boot sequence.
* **VR Controls & Input Translation:** **Not yet implemented.** Virtual Reality controller mapping (6DoF tracking, trigger inputs, grip sensors, thumbsticks) and haptic feedback translation layers from Android OpenXR/OVR input structs to Windows OpenXR input actions are currently unmapped. Inputs are either ignored or stubbed.
* **Rendering & Display:** Frame buffer capture and stereo swapchain presentation are stubbed out; visual rendering output is disabled in this preview build.

* <img width="809" height="516" alt="image" src="https://github.com/user-attachments/assets/edc755ee-c985-4f32-bbd8-cb8521f2e87d" />

