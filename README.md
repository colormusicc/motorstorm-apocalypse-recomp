# MotorStorm Apocalypse — PC static recompilation (work in progress)

An open, community port of **MotorStorm: Apocalypse** (PS3, `BCUS98242`) to Windows, built on
[ps3recomp](https://github.com/sp00nznet/ps3recomp) — the game's PPU code is statically recompiled
to native x86-64 C++, SPU programs are lifted or interpreted, and RSX is translated to Direct3D 12.

> **This repository contains no game code or data.** You need your own copy of the game,
> dumped and decrypted yourself. Lifted/recompiled output, ELFs, firmware modules, `.psarc`
> archives and logs are all `.gitignore`d and must never be committed.

## Progress: ~15 %

The game boots and runs through the front-end intro screens, but you can't race yet.

| Area | State |
|---|---|
| PPU recompilation | ✅ Whole EBOOT lifted: ~51 000 functions, 4.8 M lines of C++, builds with 0 errors |
| Boot / OS / HLE | ✅ Boots, threads, sync, filesystem (VFS), `.psarc` + patches load, saves path opens |
| Firmware LLE | ✅ `libsre` (SPURS), `libfont`, `libfontFT`, `libfreetype` run as recompiled LLE modules |
| SPU | 🟡 SPURS kernel, policy modules, edgezlib, MLAA and job modules lifted; Multistream DSP plugins run in the SPU interpreter. Havok (`hka`/`hkp`, multi-segment) not lifted |
| Graphics (RSX → D3D12) | 🟡 Menus/text/logos render; offscreen targets fixed; 3D scene reaches a GPU target but the final composited frame is mostly empty/black. MLAA path wastes ~60 ms/frame |
| Video (cutscenes) | 🔴 DivX (MPEG-4 ASP) not decoded — cutscenes play through but the picture is white |
| Audio | 🔴 AC-3 / MP3 not decoded (silence); crackling in Multistream |
| Input | 🟡 Pad works (scripted input used for automated runs) |
| Gameplay | 🔴 Never reached the main menu or a race yet |
| Performance | 🔴 ~7 fps in menus (target: 60+). Main thread waits on MLAA SPU jobs |

### How far it gets today

Epilepsy warning → **PLAY / PLAY IN 3D** menu → auto-save notice → `evologo` and `maintitles`
cutscenes (white picture, no sound) → **"MotorStorm Apocalypse After Party"** announcement
screen with *✕ CONTINUE*. That's where testing stopped — nobody has pressed past it yet.
No crashes (`FAULT 0`, `D3D12-LOST 0`) on 10-minute runs.

## Where help is needed (roughly by priority)

1. **Go past "After Party" to the main menu** and fix whatever new unresolved NIDs/stubs show up.
2. **Frame output + MLAA.** The game renders the scene on the GPU, blits it to main memory,
   runs MLAA on SPU and blits back. We don't read the GPU target back, so that path works on
   garbage. Options: (a) GPU→guest readback, or (b) composite from the GPU target directly
   (`GCM_RT_BLIT_COMPOSITE=1`, done) **and** short-circuit the MLAA job (not done — candidates
   `func_007E8854`, `func_007E99A0`, `func_007E89A4` wait on `[obj+0x2C]`).
3. **Why the composited scene is nearly empty** (lighting/compose pass vs. game intentionally dark).
4. **Video:** DivX via FFmpeg (`AV_CODEC_ID_MPEG4`) in `cellVdec`; **audio:** AC-3/MP3 in `cellAdec`.
5. **Havok SPU** images (multi-segment ELFs) lifting.
6. **Performance** in general (profiling tools are already wired in, see NOTES).
7. Make build scripts portable (they currently hard-code `C:\msa-port` paths).

## Repository layout

| Path | What |
|---|---|
| `port/` | The game project: `main.cpp`, `game_fixes.cpp` (title-specific hooks/HLE), `prx_modules.cpp` (LLE firmware), `diag.cpp`, `CMakeLists.txt` |
| `patches/` | Changes to ps3recomp, `0001…0019` in order (= `ALL-repo-changes.patch`) |
| `tools/` | Build/run scripts, ELF survey, SPU image extraction/lifting, VFS builder, log symbolizer |
| `NOTES.md` | **Full engineering log** (in Russian): every bug found, how it was verified, env-vars, current blockers. Start with the section «ГДЕ СТОИМ» at the end |

## Building (Windows)

Requirements: Visual Studio 2022 (MSVC tools), LLVM/clang-cl, CMake + Ninja, Python 3.12+.

1. Clone ps3recomp into `repo/` at commit `4e8f3ca` (v0.18.0) and apply the patches:
   ```
   git clone https://github.com/sp00nznet/ps3recomp repo
   git -C repo checkout 4e8f3ca
   git -C repo apply ../patches/ALL-repo-changes.patch
   ```
2. From **your own** copy of the game, get a decrypted `EBOOT.elf` into `elf/` and decrypted
   copies of `libsre`, `libfont`, `libfontFT`, `libfreetype` `.sprx` into `firmware/`
   (e.g. RPCS3 → Utilities → Decrypt PS3 Binaries, on copies).
3. Analyse and lift (see `NOTES.md` for the exact commands used):
   `repo/tools/ppu_loader.py`, `repo/tools/find_functions.py`, `repo/tools/ppu_lifter.py --code-end 0xF52D28 --chunks 64`
   → `recompiled/`; SPU images via `tools/extract_spu_images.py`, `tools/lift_spurs_pms.py`;
   firmware via `tools/lift_libsre.sh` / `tools/lift_prx_module.sh`.
4. `tools\build_port.bat` (edit the paths at the top first) → `build/port/MotorStormRecomp.exe`.
5. `tools/make_vfs.sh` builds `vfs/` from your disc dump, game update and RPCS3 `dev_flash`,
   then `tools/run_port.sh 300` runs it with a time limit and writes a log.

## Legal

This project is not affiliated with Sony Interactive Entertainment or Evolution Studios.
MotorStorm is a trademark of its respective owners. No copyrighted game assets or code are
distributed here; you must own the game. Source code in this repository is MIT-licensed.

## Credits

- [ps3recomp](https://github.com/sp00nznet/ps3recomp) by sp00nznet and contributors (MIT)
- RPCS3, used as the reference for correct behaviour

---

### Кратко по-русски

Открытый порт MotorStorm Apocalypse на ПК через статическую рекомпиляцию (ps3recomp).
Готовность ~15 %: игра грузится, проходит предупреждение, меню 2D/3D, ролики (без картинки
и звука) и доходит до экрана «After Party». До главного меню и гонок пока не дошли, ~7 кадров/с.
Подробный журнал работы — `NOTES.md`. Файлов игры в репозитории нет. Помощь приветствуется!
