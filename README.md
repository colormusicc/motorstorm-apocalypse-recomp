<div align="center">

<img src="assets/icon.png" width="128" height="128" alt="MotorStorm Apocalypse Recomp icon">

# MotorStorm Apocalypse · PC Recomp

**An open, community-driven native PC port of *MotorStorm: Apocalypse* (PS3)**<br>
static recompilation · no emulator · Direct3D 12

[![Status](https://img.shields.io/badge/status-work_in_progress-ff6a00?style=for-the-badge&labelColor=0d0d0d)](#-port-status)
[![Progress](https://img.shields.io/badge/progress-~25%25-ff8a00?style=for-the-badge&labelColor=0d0d0d)](#-port-status)
[![Platform](https://img.shields.io/badge/platform-windows_x64-c2410c?style=for-the-badge&labelColor=0d0d0d)](#-building)
[![License](https://img.shields.io/badge/license-MIT-7a1505?style=for-the-badge&labelColor=0d0d0d)](LICENSE)

[![Built on ps3recomp](https://img.shields.io/badge/built_on-ps3recomp-ff6a00?style=flat-square&labelColor=1a0c05)](https://github.com/sp00nznet/ps3recomp)
[![PRs welcome](https://img.shields.io/badge/PRs-welcome-ffb000?style=flat-square&labelColor=1a0c05)](CONTRIBUTING.md)
[![Title IDs](https://img.shields.io/badge/BCUS98242_·_BCES00484-1a0c05?style=flat-square&labelColor=1a0c05)](#)

</div>

> [!IMPORTANT]
> **This repository contains no game code or data.** You need your own copy of the game,
> dumped and decrypted by yourself. Lifted output, ELFs, firmware modules, `.psarc` archives
> and logs are `.gitignore`d and must never be committed.

---

## 🔥 What is this?

The game's PowerPC (PPU) code is **statically recompiled** into native x86-64 C++ with
[ps3recomp](https://github.com/sp00nznet/ps3recomp). SPU programs are lifted to C or run in an
SPU interpreter, firmware libraries run as recompiled LLE modules, and RSX graphics are
translated to **Direct3D 12**. The result is a regular Windows `.exe`, not an emulator.

## 📊 Port status

<div align="center">
<img src="assets/progress.svg" width="100%" alt="Port status: about 25%">
</div>

### 🏁 Latest progress: it loads into a race

> Community update from **[@Killeboi](https://github.com/Killeboi)** (EU build `BCES00484` 1.06, credits to **Shad**):
>
> - ✅ **Menus run smoothly**
> - ✅ **Loads into a race**: the race start audio plays, and the "track loaded" audio plays too (very laggy)
> - ❌ **Freezes on track load**, so no driving yet
> - ⬛ The menus are 3D scenes with cartoon art drawn on top. In the recomp the 3D part is still **black**
>
> Those changes aren't in this repository yet. They'll be merged once Killeboi uploads their files.

### Subsystems

| | Area | State |
|:-:|---|---|
| 🟢 | **PPU recompilation** | Whole EBOOT lifted: ~51 000 functions, 4.8 M lines of C++, builds with 0 errors |
| 🟢 | **Boot / OS / HLE** | Threads, sync, VFS, `.psarc` and game updates, saves. `cellFs` rename fixed for Windows (patch 0018) |
| 🟢 | **Firmware LLE** | `libsre` (SPURS), `libfont`, `libfontFT`, `libfreetype` run as recompiled LLE. Font em-scale fixed (0019) |
| 🟡 | **SPU** | SPURS kernel, policy modules, edgezlib, MLAA and job modules lifted. Multistream DSP runs in the interpreter. Havok (`hka`/`hkp`) not lifted |
| 🟡 | **Graphics (RSX → D3D12)** | 2D menus, text and logos render. Offscreen targets fixed. The 3D scene behind menus is black, and the MLAA path wastes ~60 ms/frame |
| 🟡 | **Video** | DivX → FFmpeg MPEG-4 path added (patch 0017, needs FFmpeg DLLs). Picture not verified with the intros on yet |
| 🔴 | **Audio** | AC-3 / MP3 not decoded (silence). Multistream crackles and lags |
| 🔴 | **Gameplay** | Reaches race loading, freezes on track load |
| 🔴 | **Performance** | Smooth in menus on the EU build, ~7 fps in this tree. Main thread waits on MLAA SPU jobs |

<details>
<summary><b>Boot sequence (US build, this tree)</b></summary>

<br>

Epilepsy warning → **PLAY / PLAY IN 3D** → auto-save notice → `evologo` + `maintitles` cutscenes →
**"MotorStorm Apocalypse After Party"** screen with *✕ CONTINUE*.
No crashes (`FAULT 0`, `D3D12-LOST 0`) on 10-minute runs.

</details>

## 🛠️ Where help is needed

| # | Task | Notes |
|:-:|---|---|
| 1 | **Fix the freeze on track load** | Top priority: it's the only thing between us and driving |
| 2 | **Black 3D menu scenes** | The scene reaches a GPU target, but the composited frame is mostly empty. Lighting/compose pass? |
| 3 | **Frame output + MLAA** | Either read the GPU target back to guest memory, or composite from the GPU (`GCM_RT_BLIT_COMPOSITE=1`, done) **and** short-circuit the MLAA job (candidates `func_007E8854`, `func_007E99A0`, `func_007E89A4` wait on `[obj+0x2C]`) |
| 4 | **Audio** | AC-3 / MP3 in `cellAdec`, Multistream lag |
| 5 | **Video** | Verify the DivX picture with patch 0017 |
| 6 | **Havok SPU** | Lift the multi-segment `hka` / `hkp` ELFs |
| 7 | **Portable build scripts** | They hard-code `C:\msa-port` paths right now |

Read [`CONTRIBUTING.md`](CONTRIBUTING.md) first, then dig into [`NOTES.md`](NOTES.md).

## 📁 Repository layout

```text
port/       the game project: main.cpp, game_fixes.cpp (title hooks/HLE),
            prx_modules.cpp (LLE firmware), diag.cpp, CMakeLists.txt
patches/    changes to ps3recomp, 0001…0023 in order (= ALL-repo-changes.patch)
tools/      build/run scripts, ELF survey, SPU extraction + lifting, VFS builder, symbolizer
assets/     README artwork
NOTES.md    full engineering log: every bug, how it was verified, env vars, blockers
            (mostly Russian; start with «ГДЕ СТОИМ» near the end)
```

## 🧱 Building

**Requirements:** Windows · Visual Studio 2022 (MSVC tools) · LLVM `clang-cl` · CMake + Ninja · Python 3.12+ · optional: FFmpeg shared DLLs for video

<details open>
<summary><b>Step by step</b></summary>

<br>

**1. ps3recomp + patches**

```bash
git clone https://github.com/sp00nznet/ps3recomp repo
git -C repo checkout 4e8f3ca          # v0.18.0
git -C repo apply ../patches/ALL-repo-changes.patch
```

**2. Your own game files.** From **your own** copy, put a decrypted `EBOOT.elf` into `elf/` and
decrypted copies of `libsre`, `libfont`, `libfontFT`, `libfreetype` (`.sprx`) into `firmware/`
(e.g. RPCS3 → *Utilities → Decrypt PS3 Binaries*, run on copies).

**3. Analyse and lift** (exact commands are in `NOTES.md`)

```bash
python repo/tools/ppu_loader.py      elf/EBOOT.elf -o analysis/loader
python repo/tools/find_functions.py  elf/EBOOT.elf --json \
       --seed-json analysis/loader/EBOOT.functions.json -o analysis/functions.json
python repo/tools/ppu_lifter.py      elf/EBOOT.elf --functions analysis/functions.json \
       --hle-stubs analysis/loader/EBOOT.imports.json \
       --code-end 0xF52D28 --chunks 64 --output recompiled/
```

SPU: `tools/extract_spu_images.py`, `tools/lift_spurs_pms.py` · firmware: `tools/lift_libsre.sh`, `tools/lift_prx_module.sh`

**4. Build**: edit the paths at the top of the script first.

```bat
tools\build_port.bat        &rem → build\port\MotorStormRecomp.exe
```

**5. Run**

```bash
tools/make_vfs.sh           # builds vfs/ from your disc dump, update and RPCS3 dev_flash
tools/run_port.sh 300       # runs with a time limit and writes logs/port-<stamp>.log
```

</details>

## 🙌 Credits

- **[ps3recomp](https://github.com/sp00nznet/ps3recomp)** by sp00nznet and contributors (MIT)
- **[@Killeboi](https://github.com/Killeboi)**: EU port, race loading, patches 0017–0023
- **Shad**: help on the EU port
- **RPCS3**: reference for correct behaviour
- Everyone in the MotorStorm community who tests, reverses and reports 🧡

## ⚖️ Legal

Not affiliated with Sony Interactive Entertainment or Evolution Studios. *MotorStorm* is a
trademark of its respective owners. No copyrighted game assets or code are distributed here;
you must own the game. Source code in this repository is MIT-licensed.

<div align="center">
<br>
<img src="assets/icon.png" width="40" alt="">
<br>
<sub>Made by the MotorStorm community · let's get it racing on PC 🏁</sub>
</div>
