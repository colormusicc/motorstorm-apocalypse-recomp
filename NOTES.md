# MotorStorm Apocalypse (BCUS98242) → Windows через ps3recomp

## Статус (актуальное — в конце, раздел «ГДЕ СТОИМ (2026-11-11)»; ниже история с 2026-11-05)

Этап разведки закончен. Полную рекомпиляцию не запускал: остановился, как договаривались.
Ничего не собрано и не запущено, поэтому ни про что нельзя сказать, что оно работает.

| Шаг | Состояние |
|---|---|
| Инструменты | Есть всё нужное, см. ниже |
| ps3recomp | Клонирован в `repo/` (HEAD `4e8f3ca`, v0.18.0), документация и шаблон прочитаны |
| Разбор EBOOT.elf | Сделан своим скриптом `tools/elf_survey.py` → `analysis/elf_survey.json` |
| Импорты против ps3recomp | Сделано, `tools/import_coverage.py` → `analysis/import_coverage.md/.json` |
| Лог RPCS3 | Разобран, выводы ниже; `analysis/syscalls_used.txt`, `analysis/hle_logged_calls.txt` |

### Сборка runtime (2026-11-05, проверено запуском)

- Пользователь разрешил запуск инструментов из `repo/`.
- `tools/build_runtime.bat [configure|build|test]` запускает vcvars64 VS 2022, затем clang-cl 23.1.4 и
  Ninja из VS. Сборка Release с `PS3RECOMP_BUILD_TESTS=ON` идёт вне дерева, в `build/runtime`.
- Результат: **188/188 целей, ошибок 0**, предупреждений много (в основном `-Wunsafe-buffer-usage`).
  Лог сборки: `build/runtime_build.log`. Итоговый артефакт: `build/runtime/ps3recomp_runtime.lib`.
- `ctest`: **21/21 passed** за 6.9 с, лог `build/runtime_tests.log`. Это только тесты, которые
  регистрирует корневой CMakeLists. Тесты из подкаталогов `tests/` в ctest не входят.
- `ppu_loader.py`: 25 338 функций по OPD, 480 импортов, entry/TOC совпадают с моим разбором.
  Вывод в `analysis/loader/`.
- `find_functions.py --seed-json`: **44 267 функций**, «every .opd descriptor address is a
  function start». Дизассемблировано 4 822 298 инструкций по всему исполняемому сегменту.
  Вывод в `analysis/functions.json`.
### PPU-лифт (2026-11-05, с согласия пользователя)

Команда:
`py -3.14 -I repo/tools/ppu_lifter.py elf/EBOOT.elf --functions analysis/functions.json
--hle-stubs analysis/loader/EBOOT.imports.json --code-end 0xF52D28 --chunks 64 --output recompiled/`

- `--code-end 0xF52D28` — конец последней исполняемой секции (секции 1–7 имеют флаг X: .init,
  .text, .fini, стабы импортов и три мелких). Всё выше — .rodata и SPU-образы.
- `--chunks 64` нужен, чтобы повторный лифт одной функции переписывал один файл.
- Время: 42 с на 16 воркерах. Лог: `analysis/lift.log`.
- Результат: **50 959 функций**. Это 43 872 найденные плюс 7 087 обёрток для входов в середину
  функции, 480 стабов импортов стали `ps3_hle_call`. 83 jump-table диспетчера.
- Объём: **65 файлов, 324 МБ, 4.83 млн строк C++** (`recompiled/`, в git не идёт).
- Непереведённые места (`TODO`): 2 742.
  - 2 655 `.word`: константы внутри кода (маски vperm `0x00010203/0x04050607…`, нули, смещения).
  - 78 `vmx_x531 v0,v17,v2`: одно и то же слово рядом с теми же масками, по-видимому данные.
  - 9 `op31_x1015/x844/x547/x1023`: стоят после безусловного `goto`, недостижимы.
  - Вывод: настоящих неподдержанных инструкций, похоже, нет. Это предположение, его проверит
    только запуск. Если игра упадёт на одном из этих адресов, первым делом смотреть сюда.
### Проект порта и SPU (2026-11-05)

- `port/` — копия `repo/templates/project`, проект переименован в `MotorStormRecomp`.
  Сборка: `tools/build_port.bat [configure|build]` → `build/port`. Python для генераторов:
  `Python314\python.exe`.
- Попытка сгенерировать SPU на этапе configure (`PORT_SPU_INPUTS=EBOOT.elf`) упала: генератор
  работает по принципу «всё или ничего» (`build/port/generated_spu/inventory.json`).
  - `hka` и 4×`hkp` (Havok): «automated lifting requires exactly one executable PT_LOAD»,
    у этих образов несколько исполняемых сегментов. Это ограничение `build_spu_workloads.py`.
  - `msngSPURS_MP3`, `edgepost-mlaa`, `edgezlib_inflate`: «unsupported instructions».
    Все `TODO spu: .word …` выглядят как маски shuffle и константы внутри кода, а не как инструкции.
  - 17 `mstream_dsp_*`: сгенерированы, но не зарегистрированы, потому что покрытие неполное.
- Решение для первой сборки: `PORT_ENABLE_SPU=OFF`. SPU-код без переведённого входа выполняет
  интерпретатор `runtime/spu/spu_interp.c`. Это медленно, но для загрузки годится.
  Потом: лифт по образам вручную (`spu_lifter.py`), отдельно разобраться с многосегментным Havok.

### Сборка порта (2026-11-05)

- `tools/build_port.bat build`, `-j 12`, Release, clang-cl: **233 шага, 0 ошибок, 8 мин 36 с**.
  Лог: `build/port_build.log` (27 443 предупреждения, все из исходников ps3recomp, в лифте `/w`).
- Результат: `build/port/MotorStormRecomp.exe`, **213 МБ**. В нём PPU-лифт, runtime, D3D12 и
  сгенерированная HLE-таблица NID по всем модулям. SPU-лифта нет.
- Первые запуски описаны в разделе ниже.

### VFS и запуск (2026-11-05)

- `tools/make_vfs.sh` собирает `vfs/` **копированием** (оригиналы только читаются). В ps3recomp
  нет защиты `/dev_bdvd` от записи, а игра делает `utime` на `ms3.psarc`.
  - `vfs/disc/PS3_GAME`: копия диска из Steam, 18 ГБ (`PS3_VFS_ROOT=vfs/disc`).
  - `vfs/dev_hdd0/game/BCUS98242`: PARAM.SFO, иконки и `ms3patchv0101…0106.psarc` из корня
    USRDIR RPCS3. sha256 сверены с оригиналами. Без `modded/`, без EBOOT, без `.edat` DLC.
  - `vfs/dev_hdd0/home/00000001`, `vfs/dev_hdd1`: пустые; `vfs/dev_flash`: копия прошивки RPCS3.
- `tools/run_port.sh <сек> [VAR=val…]` выставляет переменные VFS и `PS3_HLE_UNRESOLVED=fail`,
  запускает игру через `timeout` и пишет лог в `logs/port-<время>.log`.

**Правки порта (port/), ps3recomp не менялся:**
1. `main.cpp`: вызов `cellGame_init_from_paramsfo(<vfs>/PS3_GAME/PARAM.SFO)`. Шаблон его не
   делал, и title id был заглушкой `BLES00000`. Теперь он равен `BCUS98242`.
2. `main.cpp`: запуск сэмплера `ps3_sampler_start()` (`PS3_SAMPLE=<мс>`). В шаблоне его не было.
3. `diag.cpp` (новый): `PS3_HANG_DUMP=<с>` печатает гостевые lr/r3/sp, HLE in-flight и гостевые
   кадры на хост-стеке; `PS3_HANG_MEM=<ea>,…` показывает слова памяти; `PS3_HOST_WWATCH=<ea>` —
   watch на запись в гостевое слово через защиту страницы, ловит и хост-код. Сам watch может
   уронить процесс, это гонка single-step; годится только для диагностики.
4. `CMakeLists.txt`: `/MAP` → `build/port/MotorStormRecomp.map`; `tools/symbolize.py MAP --log LOG`
   переводит RVA в символы.
5. `main.cpp`: **заранее помечены как загруженные модули 0x17, 0x10, 0x0E, 0x02, 0x04, 0x11, 0xF035,
   0x06, 0x16** — так RPCS3 ведёт себя для импортов EBOOT (см. baseline-race.log).

**Зависание №1, исправлено (правка 5).** Главный поток крутился в `func_0063C1D4`
(спин-блокировка: `lwarx/stwcx` 0→1, 200 опросов, `usleep(250)`) на слове `0x132B6D8`,
в котором лежало `0x54`. Цепочка такая:
- `func_005BC6F0` — таблица загрузки модулей: `IsLoaded(id)`, а если модуль не загружен, `LoadModule(id)`
  и дописать id в массив `0x132B650` (счётчик в `0x132B648`, 34 места);
- в ps3recomp `IsLoaded` возвращает UNLOADED для всех модулей, поэтому дописывалось около 37 id
  и массив переполнялся до блокировки (последний id 0x54);
- в RPCS3 в массив попадает 27 id. После правки счётчик равен 0x1A (26), блокировка равна 0.
- Как нашёл: сэмплер → watchdog (`lr`) → `PS3_HANG_MEM` → `PPU_WWATCH` (запись не видел)
  → `PS3_HOST_WWATCH` → `/MAP`.

**Где загрузка стоит сейчас** (`logs/port-20261105-202230.log`, 120 с, без падения):
- Пройдено: CRT, cellGame, ~40 sysmodule, cellVideoOut, `cellSpurs Initialize "MS3"` (5 SPU),
  D3D12 (RTX 4060), sceNp/NetCtl Init, 12 PPU-потоков (`PS Button Handler`, `JobBatchHandler`,
  `EvoJobWorkerThread`, `async_1`, `fios mediathread 2/3/5/6/7`, `fios scheduler 1/4`),
  SysCacheMount `BCUS98242462406`, открыт `ms3.psarc`, прочитано его оглавление (~584 КБ).
- Стоит на SPU:
  - `CreateTask(entry=0x1178900)` — это образ **edgezlib_inflate**. Он не переведён, runtime пишет
    «async task dispatch … no lifted image … DROPPED».
  - Задания идут через `_cellSpursLFQueuePushBody` (NID **0x8A85674D**; раньше здесь ошибочно
    стоял 0x59B1EDE1), а в ps3recomp его нет.
  - `fios mediathread 5` ждёт `EventFlagWait` (flag 0x16334500), который должна выставить эта задача.
  - Workload `Wws_Job` (EvoDefault/EvoHigh): policy module по адресу `0x123B100`, 12 160 байт, не ELF,
    «PM NOT LIFTED». Сам ELF-поиск его не находит. Это SPURS job manager движка.
- Нерешённые NID на этом пути: `sys_prx_get_module_list` (0xA5D06BF0),
  `cellVideoOutGetScreenSize` (0xFAA275A4; в RPCS3 он тоже возвращает ошибку),
  `_cellSpursLFQueuePushBody` (**0x8A85674D**), `cellGameGetHomeDataExportPath` (**0x59B1EDE1**).
  (Исправлено 2026-11-05: в первой версии этих заметок NID этих двух функций были перепутаны.)

## Исходные файлы

- `elf/EBOOT.elf` — **в `C:\msa-port\elf` его не было** (как и `logs/`). Скопировал
  `C:\rpcs3\dev_hdd0\game\BCUS98242\USRDIR\EBOOT.elf`: его расшифровал RPCS3 в начале
  этой же сессии (в логе: `DECRYPT: Decrypted ...USRDIR/EBOOT.BIN -> ...EBOOT.elf`).
  PARAM.SFO обновления: `APP_VER 01.06`. sha256 ELF `68a7913d8d3c906b…`.
- `logs/baseline-race.log` — копия `C:\rpcs3\log\RPCS3.log` (5 ноября, 19:17–19:28),
  `logs/baseline-tty.log` — копия `TTY.log`. Пользователь подтвердил (2026-11-05), что это
  та самая сессия «меню → гонка → меню».
- В логе игра открывает `USRDIR/ms3patchv0101…0106.psarc` из корня USRDIR (25.5 МБ,
  оригинал), а не `USRDIR/modded/ms3patchv0106.psarc` (525 МБ). Эталон чистый.

## Инструменты

| Что | Есть | Примечание |
|---|---|---|
| clang-cl | 23.1.4, `C:\Program Files\LLVM\bin` | **не в PATH**. Для сгенерированного кода ps3recomp нужен именно clang-cl |
| CMake | 4.4.3 | в PATH |
| Ninja | только внутри VS (`Common7\IDE\CommonExtensions\Microsoft\CMake\Ninja`) | доступен из Developer Prompt; можно `pip install ninja` |
| Python | 3.14 (`py -3.14`), 3.11, 3.10 | `python` в PATH указывает на venv hermes-agent; запускать через `py -3.14` |
| Git | 2.52 | |
| Visual Studio | 2022 Community 17.14, 2026 Community 18.10, Build Tools 2022 | MSVC 14.44 / 14.51, Windows SDK 10.0.26100 |

Ставить ничего не обязательно. По удобству: добавить LLVM в PATH и поставить Ninja отдельно.

## EBOOT.elf

- ELF64 BE PPC64, type EXEC, entry OPD `0x12c89b8` → код `0x546fe0`, TOC `0x12e4c40`.
- PT_LOAD: `R-X 0x10000..0x1275468` (18.4 МБ), `RW- 0x1280000` file 0x7eb30 / mem 0x13d9b8;
  ещё TLS (`0x12e14d0`, 0x298), PROC_PARAM, PRX_PARAM (SDK 0x370001, prio 1024, stack 512 КБ).
- Секции: 35 штук, имена вычищены (shstrtab пустой). `.text` ≈ 0xf3c0b8 = **3 993 646 инструкций**.
- `.opd` (секция 25): 45 448 дескрипторов, 25 336 уникальных точек входа.
  OPD ∪ цели `bl` = **43 108 функций** (нижняя оценка, без учёта эвристик find_functions).
- VMX (opcode 4): 100 007 инструкций (2.5 %), FP: 37 390.
- Экспортов нет. Своих PRX игра не грузит: все 32 `sys_prx_load_module` идут к
  `/dev_flash/sys/external/*.sprx` через cellSysmodule.
- Импорты: **37 библиотек, 480 функций, 0 переменных**, все 480 имён найдены в логе RPCS3.
- Прямые syscalls (`sc` с `li r11,N` перед ним): 80 разных номеров; больше всего сайтов
  у 141 (sys_timer_usleep, 196 сайтов).
- Встроенные SPU ELF: **25**, все в `.rodata` основного сегмента (имена из лога RPCS3):
  - Havok: `hka` (1), `hkp` (4 варианта);
  - `spu/msngSPURS_MP3` — MP3-декодер Multistream;
  - Edge: `edgepost-mlaa-task`, `edgezlib_inflate_task`;
  - 17 DSP-эффектов Multistream (`mstream_dsp_*.pic`: reverb, eq, compressor, i3dl2, …).
  SPURS job-модули, которые не являются ELF, этот поиск не находит. Их надо ловить в рантайме
  (`SPU_DUMP_MISS`).

## Импорты против ps3recomp

Метод: свой код повторяет `gen_hle_nids.py --all` (функции cell*/sce*/sys* в `libs/<dir>/*.c`,
NID считается из имени) плюс явные `ps3_hle_register*` (hex и `ps3_compute_nid("…")`) плюс
таблицу sys_net. «Заглушка» значит, что тело пустое или в нём только `return const`
(эвристика, на двух функциях проверил вручную). «Готово» значит только, что тело нетривиальное.
Правильность это не гарантирует. Если NID не зарегистрирован, ps3recomp в рантайме молча
возвращает CELL_OK (`ppu_hle.cpp`, `PS3_HLE_UNRESOLVED=fail` превращает это в ошибку).

**Итого: 480 = 336 готово / 23 заглушки / 121 отсутствует (≈70 % покрытия).**
Гайд ps3recomp считает хорошим кандидатом игру с покрытием 95 % и выше.

| Библиотека | Импортов | Готово | Заглушка | Отсутствует |
|---|---:|---:|---:|---:|
| cellSpurs | 47 | 39 | 0 | 8 |
| sceNp | 44 | 27 | 4 | 13 |
| sysPrxForUser | 40 | 27 | 2 | 11 |
| cellHttp | 34 | 13 | 0 | 21 |
| cellSysutil | 32 | 25 | 5 | 2 |
| cellGcmSys | 29 | 25 | 3 | 1 |
| sys_net | 27 | 25* | 0 | 2 |
| sys_io (pad/kb/mouse) | 24 | 17 | 0 | 7 |
| sys_fs | 20 | 20 | 0 | 0 |
| libgem (Move) | 17 | 9 | 0 | 8 |
| cellFont | 16 | 9 | 1 | 6 |
| cellVoice | 16 | 12 | 0 | 4 |
| cellUsbd | 11 | 1 | 3 | 7 |
| cellAudio | 11 | 11 | 0 | 0 |
| sceNpTrophy | 11 | 11 | 0 | 0 |
| cellCamera | 9 | 8 | 1 | 0 |
| libvdec | 9 | 9 | 0 | 0 |
| cellAdec | 8 | 7 | 0 | 1 |
| cellJpgEnc | 8 | 1 | 0 | 7 |
| cellSsl | 7 | 3 | 0 | 4 |
| cellGame | 7 | 7 | 0 | 0 |
| libmedi (cellMediator) | 7 | 0 | 0 | 7 |
| cellJpgDec | 7 | 6 | 0 | 1 |
| cellNetCtl | 6 | 6 | 0 | 0 |
| cellSysmodule | 5 | 3 | 2 | 0 |
| остальные 12 библиотек | 29 | 15 | 2 | 12 |

\* 11 функций sys_net регистрируются только при `PS3_NET_ONLINE`.
Полный список отсутствующих по именам: `analysis/import_coverage.md`.

Отсутствующие, которые важны для оффлайн-игры:
- **cellSpurs**: `cellSpursCreateTaskset2`, `cellSpursCreateTask2`, `cellSpursJoinTask2`,
  `cellSpursTryJoinTask2`, `_cellSpursTasksetAttribute2Initialize`, `cellSpursDestroyTaskset2`,
  `_cellSpursLFQueuePushBody`, `cellSpursLFQueueDetachLv2EventQueue`. В MODULE_STATUS это
  прямо записано как «remaining SPURS surface (Taskset2)».
- **sysPrxForUser**: `sys_mempool_*` (5), `sys_prx_*` (load/get_module_info/list/register_library).
- **cellFont**: `cellFontRenderSurfaceInit/SetScissor`, `cellFontGetCharGlyphMetrics`,
  `cellFontSetupRenderScalePoint`, `cellFontGetKerning`, `cellFontEndLibrary`,
  а также `cellFontInitLibraryFreeTypeWithRevision` (FontFT). Без них не будет текста в UI.
- **cellL10n**: `UTF16stoUTF8s`, `UTF8stoUTF16s`.
- **cellSysutil и прочее**: `cellVideoOutGetScreenSize`, `cellGameGetHomeDataExportPath`
  (оба вызываются в логе), `cellAudioOutConfigure` и `cellAudioOutGetNumberOfDevice` (заглушки,
  вызываются), `cellKbRead`, `_cellGcmFunc15`.
- Остальное (HTTP, SSL-сертификаты, Mediator, JpgEnc, Photo, Usbd, Gem, NP-сообщения/друзья)
  относится к онлайну и периферии. Хватит корректных отказов в оффлайн-режиме.

Syscalls: из 80 номеров ps3recomp регистрирует 78, нет **260** и **815** (по одному сайту).

## Что показал лог RPCS3

- RPCS3 v0.0.43-20565, Vulkan, RTX 4060 Laptop, разрешение 1280x720 со scale 300 %,
  custom config `config_BCUS98242.yml` с `Write Color Buffers: true`. Фатальных ошибок (`·F`)
  нет, завершение штатное (закрыли окно в 10:39).
- Хронология: загрузка 0:59, примерно до 7:00 RPCS3 компилировал PPU/SPU-кэш, игра
  исполнялась с **7:01 до 10:39** (около 3.5 мин). Автосейвы были в 7:15, 7:42 и 8:12,
  presence менялся в 7:15 и 7:41. Где именно старт и финиш гонки, по логу точно не видно.
- **RPCS3 выполнял часть прошивки в LLE**: в тегах потоков есть `liblv2`, `libsre` (это SPURS),
  `libnet`, `libhttp`, `libssl`, `libvpost2`, `libmvcdec`. Поэтому вызовы cellSpurs,
  sysPrxForUser и сети в логе как HLE **не видны**. Колонка «видно в логе» (69 из 480) даёт
  только нижнюю границу того, что игра использует.
- Потоки и SPU: SPU thread group `MS3CellSpursKernelGroup` (5 SPU) и
  `MultistreamCellSpursKernelGroup` (1 SPU), PPU-потоки `MS3SpursHdlr0/1`, `HavokWorkerThread`,
  `Multistream PPU Update Thread`, `fios mediathread`, `PS Button Handler`, `async_N`.
- Больше всего syscalls (`analysis/syscalls_used.txt`, 128 имён): usleep (3.7 млн),
  semaphore wait/post, event_queue_receive, lwmutex, lwcond, rwlock, mutex, event_port_send.
  Синхронизацию игра гоняет очень плотно, так что производительность примитивов в рантайме важна.
- Ошибки в логе неопасные:
  - ENOENT на необязательные файлы (`Liveries/*.vsi`, `liveryeditor/liveries/*`,
    `data/timelines/*_v*`, `trophiesandrewards`, `*.edat` треков DLC);
  - `sys_mutex_lock/unlock ESRCH` у `HavokWorkerThread`, main и async. Повторяется
    3 раза за сессию, на PS3 это, видимо, тоже так. При переносе это надо воспроизводить, а не «чинить»;
  - `EDEADLK` у Multistream, `sys_net ENOTCONN` (RPCN), `cellVideoOutGetScreenSize VALUE_IS_NOT_SET`;
  - `RSX: Cannot invalidate a currently bound render target` (×3) и 845 предупреждений
    «ROP reads from hN without writing» от компилятора шейдеров.
- Файлы: игра читает `/dev_bdvd/PS3_GAME/USRDIR/ms3.psarc` и патчи из
  `/dev_hdd0/game/BCUS98242/USRDIR/`, открывает `VEHICLEPACK*/LIVERYPACK*.edat` (DLC, NPDRM),
  шрифты и кэш из `/dev_hdd1/`. Пишет сейвы, `Liveries/Cache` и кэш `/dev_hdd1`, поэтому
  нужен свой writable VFS-корень, а не `C:\rpcs3`.

## Оценка масштаба

Это AAA-проект уровня «месяцы, скорее год», а не «недели».
- PPU: около 4 млн инструкций и 43 тыс.+ функций. У flOw при 10 МБ ELF получилось 156 МБ C++,
  здесь ожидаю **300–400 МБ сгенерированного C++** и многочасовую сборку clang-cl.
  100 тыс. VMX-инструкций: каждая неподдержанная лифтером форма означает ошибку.
- SPU: 25 своих образов плюс SPURS-задачи (Havok-физика, Edge MLAA, Edge zlib, аудио Multistream).
  Без SPU не будет ни физики, ни звука, ни распаковки ассетов (edgezlib).
- RSX: игра пишет команды GCM сама (libgcm инлайнится в код игры), а пост-обработка MLAA идёт
  на SPU по кадру в памяти. Это та же причина, по которой RPCS3 нужен `Write Color Buffers`
  (предположение, не проверено).
- Самые близкие по типу порты на ps3recomp: Twisted Metal (BCUS98106) только загружает
  игровой код; inFamous дошёл до игры с LLE liblv2+libsre при ~13 fps; Killzone 3 (сообщество)
  проходит главы. Значит, это реально, но для такого класса игр ps3recomp ещё не довёл
  до готового состояния ни одну.

## Главные блокеры

1. **SPURS**: в HLE нет Taskset2/Task2 и LFQueue push, плюс Havok/Edge/Multistream как
   SPURS-задачи. Реалистичный путь тот же, что у inFamous: **LLE libsre.sprx (и liblv2)**
   из прошивки, которую пользователь установил в RPCS3 (`C:\rpcs3\dev_flash`, только чтение).
2. **Графика**: в ps3recomp рабочий бэкенд **только D3D12**. Vulkan там opt-in и только для Linux.
3. **Объём PPU-лифта** (время сборки, VMX, ошибки лифтера на 4 млн инструкций).
4. **Edge MLAA на SPU** требует согласованного framebuffer в памяти гостя (readback с GPU).
5. **cellFont** (рендер в surface): без него не будет текста в меню.
6. **DLC в EDAT** требует RAP-лицензий. Начинать стоит без DLC-паков в VFS.
7. Online (sceNp, HTTP, SSL, Mediator, RPCN): нужны корректные оффлайн-отказы, чтобы игра
   не зависала в ожидании сети.

## Решения и замечания к плану

- **Рендер: D3D12, не Vulkan.** На Windows в ps3recomp эталонный путь — D3D12
  (`docs/BUILDING.md#graphics-backends`), Vulkan там экспериментальный и только для Linux.
  Писать свой Vulkan-бэкенд значит взять на себя ещё один крупный подпроект без пользы для Windows.
- Скрипты из `repo/tools` запустить не удалось: авто-режим Claude Code блокирует исполнение
  кода из склонированного репозитория. Поэтому ELF и импорты разобраны своими скриптами в
  `tools/`, а из репо только читались исходники. Для лифта (`ppu_loader.py`, `ppu_lifter.py`)
  и сборки runtime нужно явно разрешить запуск `repo/tools/*.py` и cmake на `repo/`.
- Лог RPCS3 почти не показывает HLE-вызовы: прошивочные библиотеки работали в LLE. Для точного
  списка используемых SPURS/liblv2-функций нужна трассировка уже в порте
  (`PS3_HLE_UNRESOLVED=fail`, ftrace ps3recomp).
- `C:\msa-port` пока не git-репозиторий. `.gitignore` уже исключает elf/, logs/, analysis/,
  recompiled/, build/, repo/ и игровые форматы.

## Порядок работ (предложение)

1. Собрать runtime ps3recomp и его тесты: clang-cl, Ninja, D3D12, Release. Убедиться, что
   тесты проходят на этой машине.
2. `ppu_loader.py` → `find_functions.py --seed-json` → сравнить число функций с моими 43 тыс.
3. **[точка остановки, нужно согласие]** Полный `ppu_lifter.py --hle-stubs`, сборка
   из шаблона `templates/project`, `PORT_SPU_INPUTS=EBOOT.elf`.
4. VFS: `/dev_bdvd/PS3_GAME` смонтировать на Steam-копию (только чтение), `/dev_hdd0/game/BCUS98242/USRDIR`
   на свою папку со ссылками на оригинальные psarc (только чтение) и writable-подпапками.
   `/dev_hdd1` и сейвы держать в своей папке.
5. Загрузка с LLE liblv2 и libsre (по образцу inFamous), null-бэкенд, `PS3_HLE_UNRESOLVED=fail`:
   дойти до main loop, по пути закрывать отсутствующие NID (порядок вызова сверять с RPCS3).
6. SPU: Havok, edgezlib, Multistream. Распаковка ассетов и звук.
7. D3D12: меню (cellFont, 2D), затем трек, затем MLAA.
8. Гонка до финиша и возврат в меню; сравнение с baseline-race.log.

## Следующий шаг

### SPU: edgezlib и LFQueue (2026-11-05)

- **edgezlib_inflate переведён и работает в рантайме.** Образ извлечён из EBOOT+0x1168900
  (38 100 байт) в `spu_programs/images/edgezlib_inflate.elf`. Команда:
  `build_spu_workloads.py --images spu_programs/images --lifted port/spu --out port/spu/spu_workloads.c
  --register-fn port_spu_register_all --constructor --allow-partial`. Результат: 661 функция,
  покрытие 91,4 %, 213 `.word` (это данные). Отпечаток `0x00E729804474449A` совпадает с тем, что
  передаёт игра. Сборка: `PORT_ENABLE_SPU=ON`, `SPU_DIR=port/spu`.
  Проверено запуском (`logs/port-20261105-202908.log`): `dispatch HIT … image=1`, задача входит в
  `WAIT_SIGNAL` и ждёт.
- **LFQueue push не сделан, и это главный блокер.** Очередь игры: `q=0x16334580 size=32 depth=12
  dir=3`, то есть **ANY2ANY**.
  - Раскладка `CellSyncLFQueue` взята из RPCS3 `cellSync.h`: pop1 0x00, push1 0x08, size 0x10,
    depth 0x14, buffer 0x18 (для ANY2ANY с битом 1), bs[4] 0x20, direction 0x24, v1 0x28, init 0x2C,
    push2 0x30, hs1 0x32, pop2 0x50, hs2 0x52, eaSignal 0x70, v2 0x78, eq_id 0x7C.
  - `_cellSpursLFQueueInitialize` в ps3recomp с этим расходится: кладёт depth в 0x0C (`push1.m_h8`),
    не пишет direction/init/bs/v1/push2/pop2. Авторы сами пишут, что раскладка угадана.
  - Для ANY2ANY протокол push/pop (`GetPushPointer2`/`CompletePushPointer2`) **нигде публично не
    реализован**: в RPCS3 это пустые TODO, а сам RPCS3 гоняет здесь прошивочный libsre в LLE.
    Корректную HLE-реализацию можно сделать только реверсом. Варианты: SPU-сторона pop внутри
    edgezlib (она у нас переведена) или PPU-код `_cellSpursLFQueuePushBody` в libsre.prx.
  - Справочные исходники RPCS3 скачаны только для чтения в scratchpad, в проект не копировались.

### Решение: LLE libsre (2026-11-05, выбор пользователя)

Как это ляжет на ps3recomp (код прочитан, ничего ещё не делалось):
- `runtime/prx/prx_loader.{h,c}`: `prx_load_module(prx_module*, reg)` копирует перемещённый образ в
  гостевую память по базе B, регистрирует переведённые функции и экспорты (NID → OPD).
- `runtime/ppu/ppu_hle.cpp`: `ps3_hle_call` **сначала** вызывает `prx_resolve_export(nid)`. Если NID
  экспортирует загруженный PRX, вызов уходит в переведённый libsre, а не в HLE. Так работали
  YDKJ и flOw; `YDKJ_FORCE_HLE=<nid,…>` принудительно возвращает отдельные NID в HLE.
- В порту нужно реализовать `ps3_load_prx_modules()` (сейчас пустая в `port/stubs.cpp`).
- Конвейер: `prx_relocate.py`/`lift_prx.py --base B` → `ppu_lifter.py --raw --base B
  --symbol-prefix libsre_` → `prx_module` в порту.
- База: **0x30000000** (так у YDKJ, TOC libsre 0x30039AB0). Карта MotorStorm там свободна:
  ELF 0x10000–0x13BD9B8, TLS 0x0E000000, стек 0x0FF00000, кучи 0x10000000–0x1BA00000,
  mmapper 0x60000000, стеки потоков 0xD0000000.
- Копии `libsre.sprx` и `liblv2.sprx` из `C:\rpcs3\dev_flash\sys\external` лежат в `firmware/`
  (в git не идут). Это зашифрованные SELF (`SCE\0`). **Расшифровывает пользователь.**

### LLE libsre: сделано и проверено запуском (2026-11-05)

- Пользователь расшифровал `firmware/libsre.prx` и `liblv2.prx` в RPCS3 (ELF64 BE, type 0xFFA4, PPC64).
- `tools/lift_libsre.sh`: `lift_prx.py --base 0x30000000` (TOC 0x30039AB0, 267 экспортов в 6 библиотеках,
  импорты: 30 из sysPrxForUser и 2 из cellLibprof) → `--hle-stubs` из импортов → `ppu_lifter.py --raw
  --toc --code-end 0x3002E000 --symbol-prefix libsre_` → `prx_exports.py` (таблица NID).
  Лифт дал 940 функций, но **500 из них мусорные**, вне образа: адреса 0x0…, 0x2E5D74A0… с шагом
  0x6000, артефакт прохода tail-entry в raw-режиме. `port/prx_libsre.cpp` регистрирует только
  диапазон `[0x30000000, 0x3002E000)` (440 функций), иначе мусор перекрыл бы функции игры.
- Все 47 cellSpurs и 3 cellSync импорта игры экспортируются libsre.
- `port/prx_libsre.cpp`: `ps3_load_prx_modules()` читает образ из файла во время запуска (прошивка
  в exe не вшивается) и вызывает `prx_load_module`. `PS3_LIBSRE=0` возвращает HLE.
- SPU: `tools/extract_spu_images.py` извлекает 20 образов EBOOT (без Havok) и 2 SPU-образа libsre
  в `spu_programs/images/`, затем `build_spu_workloads.py --images … --allow-partial` переводит и
  регистрирует **22 образа**. Через `--input` образы с `.word` не регистрируются даже с
  `--allow-partial`, поэтому только `--images`.
- Инструменты ps3recomp запускать **без `-I`**: им нужен соседний модуль из `tools/`.

**Исправление №2: монтирование патчей.** Игра (`func_002B185C`) монтирует `ms3patchv0106…0101`, только
если `BootCheck` вернул `type == DISC && (attributes & CELL_GAME_ATTRIBUTE_PATCH)`. ps3recomp всегда
пишет `attributes = 0`. `port/game_fixes.cpp` переопределяет NID 0xF52639EA: вызывает оригинал и
добавляет бит PATCH, если есть `vfs/dev_hdd0/game/BCUS98242/PARAM.SFO`. Дополнительно `main.cpp` читает
PARAM.SFO обновления (APP_VER 01.06), как RPCS3. Сам по себе этот шаг патчи **не** включил;
решил бит PATCH.

**Где загрузка сейчас** (`logs/port-20261105-210039.log`, `-210357.log`):
- В порядке RPCS3: `MS3CellSpursKernelGroup` (5 SPU, **настоящее SPU-ядро SPURS**), `MS3SpursHdlr0/1`
  (PPU-обработчики libsre), cellGame, ~40 модулей, sceNp/NetCtl, PS Button Handler, JobBatchHandler,
  EvoJobWorkerThread, fios (7 потоков), патчи **0106→0101, затем ms3.psarc** (тот же порядок, что
  в baseline), async_2..4, **HavokWorkerThread**, **hkSpuUtil Helper**, ~16 очередей событий,
  `cellGcmInit` + `_cellGcmInitBody`, SetFlipMode/VBlank/UserHandler.
- Затем всё ждёт: главный поток и потоки fios в `sys_lwcond_wait` (lr 0x9BEC3C), `fios mediathread 5`
  в цикле `event_queue_receive(q=2)`. q=2 не связана ни с одним PPU-портом, события должен прислать SPU.
- На SPU ядро (образ 3) почти всё время в **интерпретаторе** (~5 млн шагов/с), горячие страницы LS
  0x0C00–0x1000 и 0x8000–0x8800. Это код, который ядро подгружает само: policy-модули (Wws_Job
  0x123B100 / 12 160 байт, taskset PM из libsre) и задачи. `SPU_DUMP_MISS` при LLE пуст:
  он срабатывает только в HLE-диспетчере.
- Прочее: `sys_spu_thread_group_disconnect_event_all_threads` (252) — заглушка; открытие файла
  с испорченным путём `'4??4??rewards.xml'` (flags 0x241, запись); нерешённые
  `sys_prx_get_module_list`, `cellVideoOutGetScreenSize`, `cellGameGetHomeDataExportPath`.

### SPURS PM как оверлеи, первые кадры, выход из-за шрифтов (2026-11-05)

**Policy-модули.** Таблица `CellSpurs.wklInfo1[]` (`0x10060600+0xB00`, по 0x20 на запись) и
`wklInfoSysSrv` (+0xD00), прочитанные через `PS3_HANG_MEM`:
- wid 0, 1 — **Wws_Job** PM `0x0123B100`, 0x2F80 (EBOOT);
- wid 2–5 — **taskset** PM `0x30023680`, 0x1E40 (libsre); taskset'ы 0x16332C00 (edgezlib), 0x14844E00,
  0x11AC0580, 0x11AC5E00;
- SysSrv — PM `0x30021480`, 0x2200 (libsre).

`tools/lift_spurs_pms.py` оборачивает каждый PM в SPU ELF по адресу 0xA00, переводит его (`--extra-funcs
0xA2C` для WWS — вектор прерывания job manager'а, как у LBP) и генерирует `port/spu/pm_register.c`:
`spu_overlay_register_verified_region(источник DMA, 0xA00, размер, образ 40/41/42, байты)`. Туда же
добавлены ELF-задачи как оверлеи: edgezlib (образ 2, src 0x01178A00 → LS 0x3000), MP3 (5) и MLAA (1).
Реестр оверлеев принимает загрузку по частям: смежная GET возвращает слот в «pending».
Проверено запуском: горячие страницы интерпретатора ушли, SPU-код работает переведённым.

`tools/run_port.sh` теперь выставляет **`SPU_INTERP_UNLIFTED=1`**: Multistream перемещает свои DSP-плагины
`.pic` в LS (0x37490, 0x38230, 0x39810…) и вызывает их. Ни один лифт под эти адреса не подойдёт,
поэтому их выполняет интерпретатор (так же сделано для FMOD в YDKJ).

**Исправления HLE в `port/game_fixes.cpp`** (ps3recomp не менялся):
- `cellGameBootCheck` — флаг PATCH (см. выше);
- `cellSslCertificateLoader` (0x571AFACA) — как в RPCS3: склейка `dev_flash/data/cert/CA%02d.cer` по битам
  flag, с перестановкой номеров для набора Baltimore. Было: всегда `required=0`, после чего HTTP-клиент
  игры падал с «Could not fetch proper cert size» и вызывался `exitspawn`;
- `cellHttpInitCookie` (0x9638F766) — `CELL_OK`.

**Где игра сейчас** (`logs/port-20261105-212342.log`, проверено запуском):
- Проходит всё описанное выше, затем Havok, GCM, **первые отрисовки**: CLEAR_SURFACE, 32 DRAW_ARRAYS
  (полноэкранный квад, prim=QUADS, фрагментная программа игры, текстура 256×256, альфа-смешивание),
  VS закэширован в D3D12. Затем cellGem, Camera, Usbd, **cellPad (XInput)**, sceNpTrophy
  (NPWR00899, contexts OK), **cellAudio** (порт 8 каналов, mixing thread), второй экземпляр SPURS
  **Multistream** (1 SPU) с плагинами в интерпретаторе, flashMP3.pic, cellHttp/cellSsl (сертификаты 7300 байт).
- Затем **штатный выход**: `cellFont OpenFontFile(/dev_hdd1/nimbussanconl-bla.ttf)` → «Cannot open
  font file» → `cellPad End`, `cellGem End`, потоки завершаются, Multistream SPU останавливается,
  libspurs пишет ассерт `handler.c:198` (это следствие остановки).
- Причина, видимо, в **cellFont ps3recomp**, который для этой игры не годится:
  1) `cellFontInitLibraryFreeTypeWithRevision` (0x7A0A83C4) отсутствует (и вся ветка FreeType);
  2) `cellFontOpenFontFile` открывает гостевой путь хостовым `fopen` без перевода. При этом файл
     игра уже скопировала в `vfs/dev_hdd1/nimbussanconl-bla.ttf`, 132372 байт;
  3) сигнатура `cellFontOpenFontFile(font, path, …)` не совпадает с SDK `(library, path, subNum, uniqueId, font)`.
- Остальные нерешённые на этом пути: `sys_mempool_create` ×4 (0xCA9A60BF), `cellUsbdRegisterExtraLdd2` /
  `UnregisterExtraLdd`, `cellMediator*` ×4, `cellHttpClientSet*` ×4 (тайм-ауты, callback), а также
  `sys_prx_get_module_list`, `cellVideoOutGetScreenSize`, `cellGameGetHomeDataExportPath`.
- Мелочь: `cellGameContentPermit` создаёт мусорный каталог `vfs/disc/game/BCUS98242/USRDIR`
  (ps3recomp делает `ensure_dirs` относительно VFS root).

### Шрифты через LLE, первый кадр игры (2026-11-05)

- Пользователь расшифровал `libfont`, `libfontFT`, `libfreetype` (ELF, 0xFFA4).
- `tools/lift_prx_module.sh <name> <base>` — обобщённый конвейер (вместо `lift_libsre.sh`). Базы:
  libsre 0x30000000, libfont 0x30100000, libfontFT 0x30200000, libfreetype 0x30300000.
  Переведено: 178, 235, 1586 функций; после фильтра по коду регистрируется 178, 185, 866.
- `port/prx_modules.cpp` (вместо `prx_libsre.cpp`) — таблица модулей. `PS3_LLE=<список>|none`,
  `PS3_LLE_DIR`. CMake собирает `recompiled_prx/<m>/` для `LLE_MODULES`, у каждого модуля свой include dir.
- Проверено запуском: нерешённого `cellFontInitLibraryFreeTypeWithRevision` нет; игра копирует и
  открывает `nimbussanconl-bla`, `japanese`, `nimbussanst-regularcondensed`, `nimbussandot-blaconita`
  (как RPCS3 в 7:07–7:11), проверяет DLC `.edat` (как в 7:14), обходит `data/timelines/*`, создаёт
  кэш `/dev_hdd1/chimp` (114 МБ) и **больше не выходит сама**.
- `sys_mempool_*` (create/destroy/free/try_allocate/allocate/get_count) — HLE в `port/game_fixes.cpp`
  по образцу RPCS3. Проверено: 4 пула создаются (4/4/64/4 блока).
- **Первый кадр игры**: `RSX_CAP=start:count:stride RSX_CAP_DIR=…` пишет BMP заднего буфера и манифест;
  `tools/bmp2png.py` переводит BMP в PNG. На кадрах 912–1800 — **экран загрузки MotorStorm**
  (эмблема в центре, текстура игры 256×256, её FP 0x00496741, одна отрисовка за кадр).
- Пользователь слышит **звук с треском**. Звук выводится; треск, видимо, от интерпретируемых DSP-плагинов
  Multistream. Задача на потом.

**Где стоит сейчас** (`logs/port-20261105-215508.log`, 8 минут):
- Игра остаётся на экране загрузки; к 8 минутам набрано около 1800–2400 кадров.
- За весь прогон всего **54 чтения файлов**, почти все в начале: потоковая загрузка не идёт.
- Потоки fios (tid 8–14) ждут в `sys_lwcond_wait` (lr 0x9BEC3C).
- Задача edgezlib (taskset 0x16332C00) **больше 40 000 раз** входит в `WAIT_SIGNAL` с `ran=0ms`;
  сообщение печатает HLE `runtime/spu/spu_workload.c`.
- Гипотеза: **гибрид HLE/LLE**. Задачи и LFQueue создаёт настоящий libsre, но системные вызовы
  SPU-задачи (WAIT_SIGNAL и др.) перехватывает HLE-механизм ps3recomp со своим состоянием сигналов.
  Сигнал, который libsre шлёт после `_cellSpursLFQueuePushBody`, до этого ожидания не доходит
  (или очередь пуста с точки зрения SPU-стороны), и распаковка не стартует.
- Следующий шаг: выяснить, где в `spu_workload.c`/`spu_channels.c` перехватывается WAIT_SIGNAL при LLE
  и можно ли отдать его настоящему taskset PM (образ 41). Проверить, видит ли SPU-сторона
  LFQueue записи (дамп `q=0x16334580` до и после push).

### Исправлен HLE-перехват taskset PM → экран предупреждения с текстом (2026-11-05)

**Причина «вечной загрузки».** ps3recomp перехватывает переход SPU на LS `0xA70` (точка входа
системных вызовов taskset PM) и подставляет HLE (`spu_spurs_taskset_syscall` →
`spu_taskset_wait_signal`). Делать это он должен только тогда, когда настоящего PM в LS нет. Две ошибки:
1. **runtime** (`runtime/spu/spu_channels.c`, проверка `lifted`): смотрела только основной образ,
   `resident_task` и `resident_ovl`, но не слоты `resident_code[]`, где живёт PM, загруженный LLE-ядром
   как проверяемый оверлей. Исправление: дополнительно `spu_lifted_lookup(ctx, 0xA70) != NULL`.
   Плюс отладочная печать по `SPU_A70_DEBUG=1`. Обе правки в
   **`patches/0001-spu-taskset-pm-syscall-resident-region.patch`** — первая и пока единственная правка ps3recomp.
2. **порт** (`tools/lift_spurs_pms.py`): span оверлея PM включал данные. В `0x2700` лежит
   `SpursTasksetContext`, его DMA-ят при каждом запуске задачи, и `spu_overlay_note_get` сбрасывал
   резидентность PM. Теперь span — только код (`CODE_END['pm_taskset'] = 0x2700`, span 0x1D00).
- Проверено `SPU_A70_DEBUG=1`: `lifted=1 standalone=0 slot0{img=41 lsa=0xA00 size=0x1D00 vs=2}
  slot1{img=2 lsa=0x3000 …}`; HLE `WAIT_SIGNAL` больше не вызывается (0 строк вместо ~26 000).

**Результат** (`logs/port-20261105-221740.log`, 4 мин, `analysis/rsxcap4/`):
- Кадры 300–1800: значок загрузки. **Кадр 2100: экран предупреждения о здоровье** — эмблема и текст,
  отрисованный шрифтами игры через LLE libfont/libfreetype (атлас глифов 2048×1024, FP 0x003A7441,
  10 отрисовок). Скриншот: `analysis/first_warning_screen.png`.
- Счётчик чтений `[fs] read` по-прежнему 54: видимо, дальнейшее чтение идёт через пути без этого лога
  (fios/psarc). По нему больше нельзя судить о прогрессе загрузки, лучше смотреть по кадрам.
- В RPCS3 предупреждение появляется примерно через 14 с после старта, у нас через 2–4 минуты
  (кадр ~2100). Это **очень медленно**. Кандидаты: SPU-интерпретатор (DSP Multistream), HLE-ожидания,
  сборка Release с `-O1` в лифте, логирование.

### Длинный прогон: нестабильный сбой Multistream (2026-11-05)

- `logs/port-20261105-222345.log` (15 мин): ни одного кадра ≥2400 не сохранено. В самом начале (строка ~1670)
  SPU-поток Multistream `0x2005` остановился: `stopped (status=0x1 pc=0x00A70 code=0x0) FAULT`, затем ассерт
  libspurs `handler.c:198`. В 4-минутном прогоне 221740 этого не было: сбой **нестабильный**.
- Экземпляр SPURS Multistream `0x100FE480`: `wklInfo1[0]` = taskset PM `0x30023680`/0x1E40, taskset
  `0x0137A880`; SysSrv `0x30021480`/0x2200. То есть это обычный taskset.
- Ход событий: ядро (`0x6C0`) прыгает на `0x16470`. Это точка возобновления задачи MP3 (код 0x3000–0x1A3F0),
  восстановленной после переключения контекста. Оверлей MP3 не резидентен, потому что контекст задачи
  восстанавливается DMA из области сохранения, а не из ELF-источника, и сверка его не узнаёт. Код идёт
  через интерпретатор (`SPU_INTERP_UNLIFTED`). Затем системный вызов на `0xA70`, где в LS **нули**
  (PM на этом SPU нет), отсюда stop(0) и FAULT.
- Гипотезы: (а) на этом SPU PM ни разу не был загружен ядром, потому что задача стартовала по пути
  «resume», а HLE раньше маскировал это перехватом 0xA70 (до патча 0001 вызов уходил в HLE);
  (б) LS контекста переинициализируется. Нужно проверить, был ли GET `0x30023680 → 0xA00` на SPU `0x2005`.
- Вариант обхода: для `0xA70` при **отсутствии** кода в LS (слово 0) всё же отдавать вызов HLE, то есть
  `standalone`, если `LS[0xA70] == 0`. Это безопасно: настоящего PM там всё равно нет.

- **Повторный прогон с обходом `a70_empty`** (`logs/port-20261105-224325.log`, 15 мин, `analysis/rsxcap6/`):
  сбой Multistream повторился (строка 1642, `pc=0x00A70 status=0x1 FAULT`), то есть обход **не сработал**.
  Все кадры с 2400 по 10950 показывают только значок загрузки (1 отрисовка); до экрана предупреждения
  игра не дошла. Похоже, без Multistream загрузка не идёт: игра ждёт звуковой движок.
- Почему обход не сработал (предположение): остановку даёт не перехват `0xA70` в `spu_channels.c:2666` и не
  `spu_interp.c`, а ветка «synthesised stop» ниже в том же диспетчере (`!fn` → слово по адресу 0 → `stop` →
  `spu_halt`), либо другой путь. **Завтра первым делом:** найти, какой код печатает `stopped (status=0x1 …
  FAULT)` для этого потока, и поставить обход там; затем выяснить, почему taskset PM не загружен на SPU
  Multistream (был ли GET `0x30023680 → 0xA00`).
- Промежуточный вывод: экран предупреждения в прогоне 221740 появился, когда сбоя Multistream не было.
  Без этого сбоя игра продвигается, хоть и медленно.

### 2026-11-06: сбой Multistream разобран

- Отладка `SPU_A70_DEBUG=1` (обе ветки, переведённая и интерпретатор) на SPU Multistream `0x2005`:
  `w=20859000 img=3 sc=0xA70 lifted=1 standalone=0 taskset@27BC=0137A880 lr=0x16470`. То есть **настоящий
  taskset PM на месте**, и вызов идёт в него. Поток останавливается уже внутри PM (`stop`, code 0), пока
  задача MP3 частично выполняется интерпретатором (её оверлей не резидентен после восстановления контекста).
  Обход `a70_empty` поэтому и не срабатывал. Сбой стал воспроизводиться 3/3.
- Решение (в том же патче 0001): **`SPU_TASKSET_FORCE_HLE=<EA,…>`**, тасксеты, чьи вызовы с 0xA70 всегда идут в HLE,
  даже если настоящий PM резидентен (`spu_taskset_force_hle()` в `spu_channels.c`, используется в обеих ветках).
  `tools/run_port.sh` выставляет `SPU_TASKSET_FORCE_HLE=0137A880` (taskset Multistream, статический адрес
  в BSS EBOOT). Загрузчик edgezlib (taskset 0x16332C00) остаётся на настоящем PM.
- Проверено: 2/2 прогона по 60 с — сбоев 0, ассертов 0, Multistream обслуживается HLE.
- Мелочь: Python-правка записала `spu_channels.c` с CRLF, и патч раздулся до 6733 строк. Вернул LF; патч
  `patches/0001-…` теперь 114 строк (+73/−3). Править исходники ps3recomp только с сохранением LF.

### 2026-11-06: загрузка не зависает, а крутится вхолостую

- Прогон 10 мин с HLE для Multistream: сбоев нет, но на кадрах 1500–7350 только значок. A/B по 4 минуты
  (HLE и настоящий PM для Multistream) — предупреждения нет ни там, ни там. То есть удачный 221740 был
  **случайностью**.
- Очередь edgezlib при зависании (`q=0x16334580`): pop=push=9, то есть всё забрано; буфер — дескрипторы
  `{src, dst(0x165B379F/0x165C379F), сжатый размер, 0x10000}`; флаг `0x16334500` events=0.
- SPU→PPU события работают (`SPU_EVT_TRACE=1`, добавлено в `lv2_register.c`, патч): libsre привязывает
  порты через syscall 251, а `throw_event` с кодом 81 на порт 17 группы 0x1000 доходит до **очереди 2** (fios),
  result=0.
- Сэмплер: 96,5 % ntdll, гостевой код 0,8 %, то есть всё ждёт.
- `SPURS_TRACE=1` за 60 с: `_cellSpursLFQueuePushBody` ×10870 (все rc=0), `cellSpursEventFlagClear` ×10870,
  `cellSpursEventFlagWait` ×11275; события готовности идут каждые ~10 мс. Цикл «push → clear → wait»
  работает примерно 180 раз в секунду.
- `PS3_FSLOG_READS=9` (ms3.psarc): всего ~105 чтений по 64 КБ за минуту, смещения не повторяются.
  **Заявок на распаковку в 100 раз больше, чем прочитанных блоков**: либо данные распаковываются
  повторно (результат неверный, и загрузчик перезапрашивает), либо источник — другие fd (патчи 3–8,
  `/dev_hdd1/chimp`).
- Следующий шаг: (1) посчитать чтения патчей и chimp (`PS3_FSLOG_READS=3..8,14`); (2) проверить
  корректность edgezlib: взять один дескриптор, распаковать его src на хосте (zlib/raw deflate) и сравнить
  с dst после завершения. Если не совпадает, искать ошибку в переводе SPU edgezlib. Если совпадает,
  искать, почему загрузчик повторяет запросы.

### 2026-11-06: настоящая причина «вечной загрузки» — флаг событий edgezlib

- Заявка edgezlib: `{src, dst, srcSize, 0x10000, 0x163347AD, flag 0x16334500, bit 0x0001, 0}`. Распаковка
  **на месте**: сжатый блок лежит в хвосте 64-КБ выходного буфера (`src + srcSize == dst + 0x10000`),
  буферов два, они чередуются, адреса нечётные.
- Цикл в fios (lr 0x99DCCC/0xE7D0D4/0xE7D278): `_cellSpursLFQueuePushBody` → `cellSpursEventFlagClear` →
  `cellSpursEventFlagWait`.
- **`cellSpursEventFlagWait` (LLE libsre) возвращает `0x8041090A` (`CELL_SPURS_TASK_ERROR_BUSY`) в 98 случаях
  из 101** (`SPURS_TRACE=1`). Игра код не проверяет и забирает буфер до готовности, получает мусор и
  перезапрашивает. Отсюда «вечная загрузка» и случайный успех в 221740.
- Флаг при зависании: `events=0, spuTaskPendingRecv=0, ppuWaitMask=0x0001, ppuWaitSlotAndMode=0x01,
  ppuPendingRecv=0, spuPort=0x11(17), direction=1(SPU2PPU)`. Один PPU зарегистрирован ожидающим, а SPU-сторона
  его не «удовлетворила». В RPCS3 BUSY возвращается при `ppuWaitMask || ppuPendingRecv`. Вероятно, теряется
  обновление при одновременной правке строки флага PPU (`lwarx/stwcx`) и SPU (`GETLLAR/PUTLLC`), то есть
  проблема атомарности PPU↔SPU в ps3recomp. Не доказано.
- Остальное в цепочке работает: SPU-события (`throw_event` code 81 → порт 17 → очередь 2) доходят,
  примерно 1 событие на заявку (4828 событий на ~4500 заявок за 40 с).
- Инструменты (порт): `PS3_HANG_SAVE=<ea>:<len>:<path>;…` (`diag.cpp`, снимок памяти в файл);
  `PS3_ZLIB_CHECK=1` (`prx_modules.cpp`): обёртки push/wait через модуль-перехватчик (база 0x30F00000,
  экспорт того же NID; побеждает последний загруженный), сжатые байты распаковываются через stb_image
  inflate и сравниваются с буфером после ожидания. Важно: вызывая переведённую функцию из своего кода,
  нужно потом догнать `g_trampoline_fn` (`run_lifted`), иначе оригинал «вернётся» до конца работы.
  Сама сверка пока бесполезна: из-за BUSY ожидание возвращается раньше готовности.
- Обход в проверке: **`YDKJ_FORCE_HLE=373523D4`**. `cellSpursEventFlagWait` уходит в HLE ps3recomp, который
  опрашивает биты `events` каждые 2 мс и не регистрирует `ppuWaitMask`, так что хрупкого рукопожатия нет.

### 2026-11-06 (продолжение): точечный HLE для ожидания и точки возврата SPU-задач

- Глобальный `YDKJ_FORCE_HLE=373523D4` (4 мин): экран предупреждения на кадре 1800 (`analysis/cap_hle/last.png`).
  Но в 12-минутном прогоне игра встала: `Multistream PPU Update Thread` (tid 27) завис в HLE-версии
  `cellSpursEventFlagWait`, потому что для флагов Multistream HLE-ожидание не просыпается.
- Решение: **постоянная обёртка в `port/prx_modules.cpp`** (модуль-перехватчик `lle-fix` на 0x30F00000,
  экспортирует NID 0x373523D4). В HLE ps3recomp идут только вызовы с `lr == 0x00E7D278` (ожидание
  fios/edgezlib), всё остальное — в настоящий libsre (`run_lifted`). Список адресов настраивается
  `PS3_EFWAIT_HLE_LR=<hex,…>`. `YDKJ_FORCE_HLE` из `run_port.sh` убран. `PS3_ZLIB_CHECK=1` теперь
  только добавляет обёртку push.
- Прогон 12 мин с точечной обёрткой (`logs/port-20261106-160007.log`): HLE-ожидание edgezlib срабатывает
  сразу (`waits=0`), но главный поток крутится в спин-блокировке `func_0063C1D4` (вызов из `func_00E9B97C`),
  а tid 27 ждёт в настоящем `EventFlagWait` на очереди 33 (порт 17 группы Multistream). Снова упираемся в Multistream.
- Причина нестабильности Multistream: при **возобновлении** задачи после переключения контекста её LS
  восстанавливается не из ELF, поэтому переведённый код не узнаётся, и PC (например `0x16470`) — это
  середина функции.
  1. `spu_taskset_register_task_elf(elf_ea, image, 0)` (генерируется в `pm_register.c` для MP3/edgezlib/MLAA):
     диспетчер берёт образ задачи по `TaskInfo.elf` (LS 0x2794). Проверено `SPU_TASKELF_DEBUG=1`
     (отладка добавлена в `spu_channels.c`): на SPU 0x2005 `elf@2794=0x01158A80` → образ 5.
  2. **`tools/spu_lifter_ret.py`** — обёртка лифтера с `--return-entries` (адрес возврата каждого
     `brsl/brasl/bisl/bisled` становится входом). SPU-образы переведены заново
     (`build_spu_workloads.py … --lifter tools/spu_lifter_ret.py`); старая версия в `port/spu.bak_noret`.
     Теперь в MP3 есть вход `0x16470`.
- С настоящим PM для Multistream (`SPU_TASKSET_FORCE_HLE=none`) сбой остаётся, но в другом месте: после
  интерпретируемого DSP-плагина (0x37490) задача возвращается на `lr=0x3930` (вход в переводе есть) и
  останавливается `stop` (status 1, code 0) на pc 0x3930. Не разобрано.

### 2026-11-06 (вечер): распаковка исправна; настоящая причина — кэш fios на dev_hdd1

- Порядок проверен (`SPU_DMA_RANGE=<dst>-<dst+64K>`, `SPU_ATOM_EA=16334500`): edgezlib делает DMA PUT
  результата **до** установки флага (`PUTLLC` на 0x16334500), а ожидание возвращается после. Гипотеза
  «PUT отстаёт от флага» неверна.
- Раскладка заявки (строится в `func_0099DB8C`, вызов из `func_00E7D4CC`, у которого `src+2, size-2` — пропуск
  2-байтового заголовка zlib): `+0 src, +4 dst, +8 сжатый размер, +0xC сколько байт сохранить,
  +0x10 &slot.busy|1 (слот 0x2C байт, busy на +0x28), +0x14 флаг событий, +0x18 u16 биты,
  +0x1A u16 пропуск в начале, +0x1C u16 пропуск в конце`. fios читает 64-КБ блок окнами по 4 КБ
  (пропуск 0, 0x1000, 0x2000…), и edgezlib каждый раз распаковывает весь блок, а кладёт только окно.
  Прежние «несовпадения» и «отставание на одну заявку» — ошибка моей сверки, не учитывавшей пропуск.
  Исправленная `PS3_ZLIB_CHECK=1`: **7000 из 7000 блоков совпадают** (`zcheck blocks ok=7000 bad=0`).
- **Настоящая причина «вечной загрузки»**: fios при первом запуске заполняет кэш распакованных файлов на
  `/dev_hdd1` (пишет `/dev_hdd1/chimp`, затем `cellFsRename` → `/dev_hdd1/<x>/<hash>`). У RPCS3 кэш уже
  заполнен: `C:/rpcs3/dev_hdd1/caches/BCUS98242_BCUS98242462406`, 2826 файлов, 1,8 ГБ; в baseline-логе
  всего 128 переименований. У нас `cellFsRename` (`repo/libs/filesystem/cellFs.c`) берёт путь из **своей**
  таблицы, отдельной от `runtime/ppu/ppu_fs.cpp` (open/read/mkdir), а в ней не было `/dev_hdd1` вообще
  (`/dev_hdd0` вёл в `gamedata/`). Переименование молча возвращало ENOENT, кэш не заполнялся, и `chimp`
  вырос до 114 МБ.
- Исправление в `port/main.cpp`: `cellfs_add_path_mapping("/dev_hdd0/", PS3_HDD0_ROOT)` и
  `("/dev_hdd1/", PS3_HDD1_ROOT)`. Проверено запуском: 150 с → 813 переименований, 0 ошибок; следующие 300 с →
  ещё 1803; в `vfs/dev_hdd1` теперь 2617 файлов, 1,8 ГБ (у RPCS3 2826). Кадр 1800 — всё ещё экран загрузки.
- Ещё замечено: прогон без `PS3_STDIO_UNBUFFERED=1` однажды не дошёл до кэша за 120 с (нестабильность
  Multistream или буфер вывода, не разобрано).

- Следующий прогон (480 с, `logs/port-20261106-170545.log`): ещё 209 переименований, в `vfs/dev_hdd1` теперь
  **2826 файлов — ровно как у RPCS3**, кэш заполнен. Но дальше игра тысячи раз вызывала
  `cellCameraOpenEx` (5315 раз): HLE ps3recomp по умолчанию изображает подключённую камеру, а в RPCS3
  (камеры нет) `OpenEx` не вызывается ни разу. В `tools/run_port.sh` добавлено **`PS3_CAMERA=0`**
  (готовый переключатель в `repo/libs/hardware/cellCamera.c`: `GetType` → UNKNOWN, `IsAttached` → 0).
- С `PS3_CAMERA=0` (480 с, `logs/port-20261106-171429.log`): `OpenEx` 0 раз; на кадре 1920 (~8 мин) —
  экран предупреждения с текстом (`analysis/cap_nocam/f1920.png`, 10 отрисовок). Дальше пока не смотрел.

- 15 мин (`logs/port-20261106-172302.log`) и 11 мин с профилем (`logs/port-20261106-173913.log`):
  после экрана предупреждения кадры почти не идут (за 15 мин до кадра ~1900). Профиль `PS3_SAMPLE=10`:
  96 % времени ожидание в ntdll, гостевого кода <1 % — игра **не тормозит, а стоит**. Снимок
  (`PS3_HANG_DUMP`): поток fios tid 11 ждёт флаг edgezlib (`lr=0x00E7D278`, флаг 0x16334500, HLE-ожидание
  печатает `EventFlagWait BLOCKED`); главный поток и остальные fios ждут его на lwcond. Ядро SPURS
  крутится вхолостую (`[spu-spin] pc=0xCF4 GETLLAR ea=0x10060600`) — работы для SPU нет.
  С тёплым кэшем все 40 распаковок проходят в начале, а 41-я (после предупреждения) не завершается:
  сигнал задаче edgezlib о новом запросе теряется. Это гонка: короткие прогоны до 41-й не доходят.
- Очередь событий 34 (`port_send(port=30): queue 34 FULL`) — не проблема: в RPCS3 `AmadeusDemoThread`
  тоже оставляет в ней 2 непрочитанных события.
- `cellGameGetHomeDataExportPath` (0x59B1EDE1) не было: игра брала мусор из буфера и пыталась создать
  `<мусор>rewards.xml`. Добавлен HLE в `port/game_fixes.cpp`: ответ `CELL_GAME_ERROR_NOAPP` (0x8002CB08), как
  у RPCS3 (baseline 7:04.29). Другие отсутствующие: `cellVideoOutGetScreenSize`, `cellUsbdRegisterExtraLdd2`,
  `cellMediator*`, `cellHttpClientSet*Timeout`/`SetTransactionStateCallback`, `sys_prx_get_module_list`.

### 2026-11-06 (ночь): два варианта остановки после предупреждения; система задач; lwcond

Остановка — гонка, вариантов два (оба воспроизводятся за ~5 мин с тёплым кэшем):
- **(А)** 41-я распаковка edgezlib не завершается: tid 11 в `EventFlagWait` (`lr=0x00E7D278`), SPURS простаивает.
- **(Б)** (чаще, всегда с `SPURS_TRACE=1`) главный поток крутится в `func_0063C1D4` (спинлок с
  `sys_timer_usleep(250)`), вызов из `func_00E9B97C`. Разбор:
  - Класс с vtable `0x01098A08` — «ворота» на спинлоках: +8 охрана, +0xC слово ворот;
    vtbl+0x08 = `func_00E9B97C` (ждать, пока открыто), vtbl+0x0C = `func_00E9B9DC(x, open)`.
  - Ворота принадлежат **задаче системы задач игры**: `0x639FDC` (старт задачи) закрывает их,
    `0x63A3C4` (завершение) открывает, `0x63A2A4` (ждать задачу) ждёт. Цепочка указателей:
    `this2=[0x0FEFDB9C]`, `obj=[this2+4]`, ворота `=[obj+0xC]`.
  - `PS3_GATE_TRACE=1` (`port/game_fixes.cpp`, перехват 0xE9B97C/0xE9B9DC): за весь прогон главный поток
    ставит три задачи (имена в объектах: `...T06.liv`/`ogue_prologu...`/`simulation_sun...`),
    `JobBatchHandler` (tid 5) завершает **одну**, остальные две не завершаются никогда; главный поток ждёт
    одну из них. Распределитель задач — `func_00637038` (раздача по рабочим с приоритетами). Дальше не разобрано:
    куда уходят две задачи (рабочий `EvoJobWorkerThread` tid 6 ждёт семафор id 0x14 на `0x63B8F4`, или SPU).
  - Копирование `Liveries` из RPCS3 не помогло (возвращено как было).
- Новые инструменты диагностики (`port/diag.cpp`): `PS3_HANG_CHAIN=<ea>:<+off>:...` (проход по указателям в
  снимке), `PS3_HOST_WWATCH_DELAY=<с>` (включить слежение за записью позже), снимок печатает последнюю
  ненулевую запись в отслеживаемое слово.
- **lwcond в ps3recomp был заглушкой**: `sys_lwcond_signal*` ничего не делали, `sys_lwcond_wait` = `Sleep(1)`.
  Теперь настоящий (`repo/runtime/ppu/ppu_sysprx.cpp`, `patches/0002-real-lwcond-signal-wait.patch`):
  счётчик поколений + host condition variable на каждый lwcond, ожидание до сигнала/таймаута
  (`CELL_ETIMEDOUT`), но не дольше 20 мс (страховка: пропущенный сигнал стоит задержки, а не зависания).
  `PS3_LWCOND_POLL=1` — старое поведение. Проверено: собирается и запускается; на варианте (Б) не помогло.
- Файлы патчей разделены: `0001` (SPU taskset/PM), `0002` (lwcond), `0003` (диагностика `PPU_CAS_LOCKLINE`,
  `SPU_EVT_TRACE`).

### 2026-11-06 (поздно): цепочка системы задач до рабочего EvoJob — ГДЕ ОСТАНОВИЛИСЬ

- Экран предупреждения в RPCS3 уходит сам через ~12 с (проверил пользователь), кнопку он не ждёт.
- `cellUserInfoGetStat(id=0)` (0 = текущий пользователь) ps3recomp отклонял → игра хэшировала пустого
  пользователя. HLE в `port/game_fixes.cpp` отвечает «пользователь 1, `User`». Теперь игра ищет
  `Liveries/00000001/` и сама переименовывает его в `Liveries/00000000/` (папку создал я для проверки, оставлена).
  Экран предупреждения после этого и lwcond появляется раньше (кадр ~1100 вместо ~1800). Зависание осталось.
- **Разбор задачи, которую ждёт главный поток** (вариант Б):
  - класс задачи vtable `0x01098970`; `+0x14..+0x18` — список элементов, `+0x20` — маска/приоритет
    (выполненная 0x11, зависшие 0x13 и 0x05), `+0x30` — счётчик (старшие 16 бит), `+0x43` — флаги;
  - готовность: `vtbl+0x38 = 0xE9B5F4`: `u16[+0x30] == 2`. Раздача `0x63A410` (lr `0x63A458`) делает CAS
    `0x80000000 → 0x00030000` (минус `(0x7FFE − число_элементов) << 16`), каждый готовый элемент вычитает
    `0x10000` (`0x63A3A0` → `0x63C164`). У зависшей задачи остаётся **3**: её единственный элемент
    (`[+0x14]` → объект vtable `0x01333D10`) не завершился. `PPU_CASWATCH=18171610`: других CAS нет,
    то есть это не потеря атомарного обновления;
  - «элемент готов» (`0x63A3A0`) вызывается только из `0x63BA4C` — цикла **EvoJobWorkerThread** (tid 6), а тот
    спит на семафоре **20**, который за весь прогон не постится ни разу;
  - постановка в очередь EvoJob — `func_0063B5E8` (слот `+0x10` vtable рабочего `0x010989B8`): ждёт место
    (семафор с init=128), копирует 128-байтную запись в кольцо, постит семафор рабочего (класс семафора
    vtable `0x01098A7C`: `+8` wait `0xE9BD0C`, `+0x10` post `0xE9BDA8`). Перехват
    (`PS3_GATE_TRACE=1`) показал: **`0x63B5E8` не вызывается ни разу**. Главный поток при постановке каждой задачи
    постит только семафор 18 (`lr=0x635EDC`).
  - Распределитель `func_00637038` (поток JobBatchHandler): гребёнчатая сортировка рабочих, затем для
    каждой задачи вызывает у рабочего `vtbl+0x3C` с sret (`r3 = r31+0x94`, `r4` = рабочий) и сравнивает
    ответ с `[r21+0x70]`; при совпадении раздаёт (`0x63A410`).
- **Следующий шаг:** понять, почему распределитель не отдаёт элемент рабочему EvoJob. Уже собраны перехваты всех
  слотов класса рабочего (`evo_slot_*`, лог `[gate] EVOSLOT`, тоже под `PS3_GATE_TRACE=1`), прогон не доделан. Смотреть:
  вызываются ли слоты рабочего вообще; что возвращает `vtbl+0x3C`; как маска задачи `+0x20` (бит 1/бит 2)
  сопоставляется с рабочими. Возможно, второй тип рабочего — SPU (Wws/SPURS job), и он у нас не зарегистрирован.
- Ещё один вариант остановки (В): главный поток ждёт семафор в коде Multistream (`func_007E89A0`, lr `0x67DF60`),
  до экрана предупреждения не доходит.

### 2026-11-07: аргументы на стеке в мосте LLE, SPU-рабочий Wws, модули заданий

- **Цепочка до SPU.** Главный поток при постановке задачи опрашивает рабочих (`vtbl+0x08`, «принять?»).
  Рабочих двое (`[планировщик 0x012EC2F4]+0x28`): `0x100E3BB0` (EvoJob, vtable `0x010989B8`, маска 1) и
  `0x100E7EB0` (**SPU/Wws**, vtable `0x01098900`, `+4 = 5` SPU, маска 2, `+0x10 = 0x10060600` CellSpurs).
  Таблица масок типов (`[планировщик+0x18]`) у всех типов = 2, так что задачи загрузки идут на SPU-рабочего.
  Его «сброс» `0x639798` (вызывает JobBatchHandler, lr `0x6364C0`) кладёт списки заданий в очередь
  `0x18184D80` и делает `cellSpursReadyCountStore`. Встроенная библиотека заданий `0x809000–0x80B000`
  использует `_cellSpursWorkloadAttributeInitialize` / `cellSpursAddWorkloadWithAttribute`.
- **Ошибка 1 (исправлена): аргументы 9+ терялись в мосте импорт → LLE.** `repo/runtime/ppu/ppu_hle.cpp`
  (`FLOW_REENTRY_ISO`, по умолчанию включено) сдвигает `r1` на 0x1000 перед вызовом функции libsre, считая,
  что «аргументы в регистрах». Но 9-й аргумент по ABI лежит на стеке (`r1+0x70`): `_cellSpursWorkloadAttributeInitialize`
  получала `maxContention = 0`, и у нагрузок Wws (0, 1, 6, программа `0x0123B100`) в CellSpurs `wklMaxContention`
  был 0, поэтому ядро их не запускало. Исправление: копировать 0x100 байт кадра вызывающего в новый кадр.
  Проверено: `wklMaxContention` = `05 05 08 05 05 05 01`, SPU начали брать нагрузки Wws
  (`wklCurrentContention` wkl0 = 2, wkl6 = 1). `SPURS_TRACE` теперь печатает r7–r10 и стек для этих функций.
- **Ошибка 2 (исправлена): модули заданий Wws не были переведены.** Список заданий (например `0x185A9100`) указывает
  на код в EBOOT (`0x0118A800`, «сырой» модуль: 4×`ila`, вход, размер, `0xC0DEC0DE`) и на счётчик задачи
  (`задача+0x30`, его уменьшает SPU по завершении). Переведены 39 модулей:
  `py -3.14 -X utf8 repo/tools/lift_jobmods.py --engine-elf elf/EBOOT.elf --out port/spu_jobs --symbol-prefix msjob`
  (0 неподдерживаемых инструкций), скопированы в `port/spu/jobmod_*` (с правкой `#include`) вместе с
  `jobmods_register.c` (проверяемые регионы по EA, образы 200–238). В сборке теперь 64 SPU-образа.
- **Где стоим:** задания всё равно не выполняются. `SPU_DMA_RANGE` по всему диапазону модулей
  (`01181E00-01236300`): **ни одной** загрузки кода задания за прогон, то есть ни одно задание Wws на SPU у нас
  не выполнялось. Программа Wws читает очередь (`GET 128 B` с `0x18184D80`, pc `0x2554`) и список (`GET 0x90 B`,
  pc `0x30B0`), затем уходит в ядро (`pm_wwsjob 0x3040 → 0x2480 → 0x23DC → kernel 0x814`), и ядро запускает SysSrv,
  которая ждёт события (`spu_rdch`). Почему Wws не загружает задание — следующий шаг (разбор `pm_wwsjob`
  0x2480/0x3040 после чтения списка).
- Также сделано: `cellUserInfoGetStat(id=0)` в `port/game_fixes.cpp` (раньше игра считала, что пользователя нет);
  обход потерянного пробуждения edgezlib в `port/prx_modules.cpp` (`PS3_EZ_KICK`, сигнал задаче через
  `_cellSpursSendSignal`, если флаг не пришёл за 50 мс; в последних прогонах ни разу не понадобился);
  перехваты для разбора (`PS3_GATE_TRACE=1`: ворота задач, слоты EvoJob/SPU-рабочих), `PS3_HANG_RVA=1` в снимке
  (сырые адреса стека; перевод в имена — скрипт по `build/port/MotorStormRecomp.map`).

- Дальше (2026-11-07, вечер): `PS3_PM_NOLIFT=wwsjob` (переключатель в `port/spu/pm_register.c`) оставляет программу Wws
  интерпретатору. Задания всё равно не загружаются, так что это **не ошибка перевода**, а то, что программа видит в памяти.
  Атомиков программы Wws вне заголовка CellSpurs почти нет (`SPU_ATOMTRACE`). Блок аргумента нагрузки 0
  (`0x18184C80`): `+0` PM `0x0123B100`/`0x2F80`; `+0x30` четыре очереди `0x18184D80`, `0x18185200`, `0x18185680`,
  `0x18185B00` (видимо, по приоритетам); `+0x40` `00020002 ffffffff 0001ffff 0000ffff` (счётчики/индексы очередей?);
  `+0x98…` 11 указателей `0x1816F920…` (дескрипторы SPU?); в очереди 0 два элемента `(0x00900001, 0x185A9100)` и
  `(0x00800001, 0x187AB700)`. Следующий шаг: понять по коду программы Wws (функции `0x3040`, `0x2480`, `0x37C0`,
  `0x3868`), какое поле она проверяет перед загрузкой задания, и сравнить с тем, что пишет PPU-сторона
  (`0x639798` → `0x809c98/0x80a4b8/0x80a9c4/0x809a68/0x80aaac/0x80a9fc`).

### 2026-11-08: программа Wws работает на прерываниях SPU; интерпретатор их не доставлял

- Трассировка интерпретатора (`SPU_INTERP_PCTRACE=<lo>-<hi>[,file]`, новая, `repo/runtime/spu/spu_interp.c`;
  программа Wws через интерпретатор: `PS3_PM_NOLIFT=wwsjob`). Путь: `0x3040 → 0x23C8` (атомики) `→ 0x2554` (GET очереди
  в LS `0xD00`) `→ 0x2480 → 0x30B0` (GET списка) `→ 0x2874` (выдача `GETLB` со stall-and-notify) `→ 0x3168 → 0x2CC0`,
  затем вечный цикл `0x2C38 → 0x2C48 → 0x2C58 → 0x2D00 → 0x2D08`. Цикл ждёт `loadJobState` (LS `0x12A0`, младшее
  полуслово) = 2 или 3; `0x2D00` — `bie` (окно прерываний), `0x2D08` — `bid`. Маска событий `0x2`
  (`SPU_EVMASK_LOG=1`, ставится на `0x2BA0`) — DMA-list stall-and-notify. Раскладка LS та же, что у WWS в LBP
  (комментарии в `spu_dma.h`/`spu_drain.c`/`spurs_policy.c`).
- **Ошибка 3 (исправлена): интерпретатор SPU не доставлял асинхронные прерывания.** Переведённый путь берёт
  прерывание после каждого перехода при `int_enable`, а интерпретатор не проверял никогда. Добавлено в цикл
  `spu_interp_run_until`: если `int_enable && (event_status & event_mask)` → `spu_take_interrupt` (вектор LS 0).
  `SPU_INTERP_NOIRQ=1` — старое поведение. Проверено: `[irq] TAKE srr0=0x02D08 … vector->0x00A2C`, программа Wws
  **загружает код заданий** (`DMA 0x0118A800 → LS 0x4000`, десятки модулей), главный поток ушёл с ожидания задачи.
- Новая картина: главный поток в цикле кадра `func_0067DC64` (ждёт `[obj+0x2700] == [obj+0x5C0]`, obj `0x112F9E80`,
  vtable `0x01099748`), со сном 100 мкс. Цикл идёт (запись нуля в `+0x2700` повторяется), но кадров мало: ~1350 за 10 мин.
- **Переведённая** программа Wws ведёт себя иначе: после чтения списка уходит в ядро (`0x3040 → 0x2480 → 0x23DC →
  kernel 0x814`), окно прерываний не открывает, `[irq]` нет ни разу. Пока работает только интерпретируемая
  (`PS3_PM_NOLIFT=wwsjob`). Найти расхождение — следующий шаг.
- Новые диагностики: `SPU_IRQWIN_LOG=1` (окна прерываний в `spu_drain_call`), `SPU_EVMASK_LOG=1` (уникальные записи
  маски событий), `PS3_HANG_RVA=1`. Патч: `patches/0005-spu-interp-irq-and-diag.patch`.

- Дальше (2026-11-08): 20-минутный прогон с `PS3_PM_NOLIFT=wwsjob` — кадры встают на ~1300 (экран предупреждения).
  Главный поток в `func_0067DC64` ждёт `[obj+0x2700] == [obj+0x5C0]` (obj `0x112F9E80`): выдано 4, завершено 0.
  Ненулевых записей в `+0x2700` нет ни разу (`PS3_HOST_WWATCH=112FC580` + `PS3_HOST_WWATCH_DELAY=20`). Ссылка на счётчик
  (`PS3_HANG_FIND=112FC580`, новый поиск по памяти в `diag.cpp`) лежит в структуре `0x110B1F80` рядом с адресами
  видеопамяти RSX (`0xC02ABA80`, `0xC0465E80`): похоже на постобработку кадра на SPU (MLAA: задачи `edgepost_mlaa`,
  ELF `0x01170080`, создаются из `lr 0x7E8CD8`, taskset `0x11AC…`).
- `AmadeusDemoThread` (lib `0x809BE0`) крутит `cellSpursGetWorkloadInfo(spurs, wid 6)` (140 тыс. раз) и ждёт простоя
  нагрузки 6 (`readyCount 0` и `contention 0`); ответ всегда «занята». Интерпретируемая программа Wws не отпускает SPU
  (крутит цикл `bie/bid` в ожидании прерывания), а переведённая уходит в ядро без прерывания. Нужен правильный путь
  прерываний в **переведённой** программе Wws — тогда, вероятно, решатся оба ожидания.
- Диагностика `SPU_IRQWAIT_DIAG=1` теперь печатает первые 12 случаев сразу (`spu_drain.c`), ещё не опробована.

### 2026-11-08 (день): потоки SPU ядра умирали; вопрос «переведённая Wws не берёт прерывания» снят

**Прерывания.** На текущем exe расхождения нет. `logs/port-20261108-140018.log` (переведённая, `6/6` оверлеев) и
`-121722.log` (`PS3_PM_NOLIFT=wwsjob`, `5/6`): прерывания берутся (`[spu-int] TAKEN`, 16 строк — вероятно, это лимит печати
`PS3_LOG_FIRST`, а не их число), падения на тех же адресах. Запись выше «`[irq]` нет ни разу» относится к старой сборке.
Трассировка `SPU_INTERP_PCTRACE=2C00-2D10` без `NOLIFT` дала 9985 блоков на всех пяти SPU: цикл ожидания 0x2C38–0x2D08
**исполняет интерпретатор и при «переведённой» программе**, прерывание берётся там же (`[spu-interp] interrupt taken at
LS 0x02D08`). Почему управление уходит в интерпретатор (вероятно, адрес 0x2D08 после `bie` не зарегистрирован как вход),
**не проверено**. Быстрее программа Wws от перевода на этом участке не становится.

**Ошибка 4 (исправлена): поток SPU завершался сам.** Во всех свежих прогонах (проверено 114601, 120652, 121722, 140018) все
5 SPU ядра останавливались: `stopped (status=0x1 pc=… ) FAULT`, затем ассерт libspurs `handler.c:198`, «The SPURS is aborted».
`status=0x1` — это `SPU_STATUS_RUNNING`: SPU не останавливался, драйвер потока просто вышел. Значит, вывод выше про ожидания
(`[obj+0x2700]`, `AmadeusDemoThread`) был сделан на уже мёртвом SPURS, и причины «SPU не отпускает» и «уходит в ядро» надо
проверять заново. Две причины (`patches/0006-spu-interp-rejoin-and-restart-state.patch`, `repo/runtime/spu/spu_channels.c`):
1. pc 0x41D8/0x10F38: после `spu_interp_run` в ветке `SPU_INTERP_UNLIFTED` был `return` без постановки перехода. Вложенный
   `spu_drain_call` это переживает, верхний `SPU_DRAIN` видит пустую цепочку и выходит. Теперь ставится `spu_indirect_branch`.
   Лог `[spu] interp rejoin at driver level`; `SPU_INTERP_NOREJOIN=1` — старое поведение.
2. pc 0x2864: при перезапуске (`case 2` в `spu_run_with_halt`) оставались `drain_ret_pc`/`irq_frame` брошенного цикла, и
   интерпретатор останавливался на точке возврата, у которой нет владельца. Теперь оба поля сбрасываются.

**Проверено запуском** (`logs/port-20261108-141147.log`, 240 с, переведённая Wws): `FAULT` 0, ассерта нет, отрисовок к концу
6000 (было 1000 за 150 с при падении; с первой правкой 4000 за 150 с). Кадр 1200 (`analysis/cap_rejoin/frame_03`) — значок
загрузки, одна отрисовка, FP `0x00496741`. Кадр 1500 (`frame_04`): в манифесте **133 отрисовки** (экран предупреждения давал 10),
но `backbuffer.bmp` не записан (сначала здесь стояло «оборвал `timeout`» — неверно, см. длинный прогон ниже). Контроль с
`NOLIFT` снят только с первой правкой (`-140846.log`). Звук и стабильность на длинном прогоне не проверены.

**Осторожно.**
- `spu_irqwait_diag`: глобальный лимит 36 строк съедается первыми 12 случаями на поток за секунды, поэтому отчёт раз в 2^22
  диспетчеризаций не печатается ни разу (в 141147 нет ни одной строки с большим `dispatches`). Для реальных зависаний лимит надо разделить.
- В прогоне подряд идут `[mfc] REJECTED malformed transfer pc=0x02074 size=0` и элемент списка `FFFFFFE7 FFFFFFF9` при
  `RESUME tag=5` — не разобрано, возможно, мусор в LS 0x1548.
- `port_send(port=30) queue 34 FULL -> EBUSY` повторяется, на 150-секундных прогонах это есть и до правок.

**Длинный прогон** (`logs/port-20261108-141904.log`, 1200 с, переведённая Wws, `analysis/cap_long/`):
- `FAULT` 0, ассерта libspurs нет, `non-local return` и `spu-depth` нет. Падений за 20 минут не было.
- Экран предупреждения с текстом: кадр 1500, через ~38 с после старта (время файлов манифестов; раньше было 2–4 мин, в RPCS3 ~14 с).
- Кадр 1800: 129 отрисовок и внеэкранные цели 1280×1080 (в кадре 1500 было 10). **Сначала я счёл это выходом за экран предупреждения —
  неверно**: см. раздел про DRED ниже, на экране остаётся предупреждение, а отрисовки идут во внеэкранные цели.
- **Очень медленно:** кадры 1500→1800 заняли 552 с, то есть ~0,5 кадра/с. Кадр 2100 за оставшиеся 610 с не захвачен.
- Захват: `[RSXCAP] frame 1800` напечатан, рендер после него шёл дальше (до `draw_arrays #30000`), так что захват не завис. Но ни
  `backbuffer.bmp`, ни `rt_*.bmp` нет: `cap_readback_write` молча вышла. То же в 240-секундном прогоне (кадр 1500, 133 отрисовки,
  за ~100 с до конца). Причина не выяснена; похоже, ломается на кадрах с большим числом отрисовок.
- Снимок на 1190-й секунде: главный поток `lr=0x007FB398` (`func_007E89A4` → `func_007FAEDC`), один снимок, зависанием это не считать.
- Звук не проверялся.

### 2026-11-08 (вечер): GPU зависал на первом тяжёлом кадре (`patches/0007-d3d12-deferred-release-dred-capture-errors.patch`)

Захват не был сломан: `cap_readback_write` молча выходила, потому что устройство D3D12 уже погибло
(`hr=0x887A0005 DEVICE_REMOVED`, причина `0x887A0006 DEVICE_HUNG`, `logs/port-20261108-153042.log`). После этого гость продолжал слать
отрисовки в мёртвое устройство, в логе это не было видно никак. Все выводы о «кадрах с 130 отрисовками» до этого раздела сделаны на мёртвом GPU.

**Ошибка 5 (исправлена): освобождение ресурсов, на которые ссылается открытый список команд.** `off_rt_get` при смене размера цели делал
`wait_for_gpu()` и `Release`, но `wait_for_gpu` ждёт только отправленную работу; внеэкранные цели пересоздаются внутри кадра. DRED: список
выполнен на 9–11 из ~349 операций, 0 отрисовок, зависла операция 8 (`COPYTEXTUREREGION`), ошибка страницы, «recently freed: type=34
(RESOURCE), без имени» — а текстуры и цели к этому моменту имели имена. Падало на кадрах 1413 и 1583 (момент плавает).
Исправление: `gfx_defer_release` держит ресурс до `wait_for_gpu` в начале следующего `render_frame`. То же для слотов текстур и вытеснения целей.
Лог показывает срабатывание на кадрах 1451–1452 (`[D3D12] deferred release of render target…`).

**Проверено запуском** (`logs/port-20261108-154856.log`, 300 с, переведённая Wws): `D3D12-LOST` 0, `STUCK` 0, `FAULT` SPU 0; кадры 1460–1560 записаны
(по 17 BMP, `analysis/cap_fixed/`). 20-минутный прогон с этой правкой **не делался**.

**Что на экране:** `backbuffer` кадра 1560 — по-прежнему экран предупреждения. 126–136 отрисовок идут во внеэкранные цели (сцена готовится за
экраном). `rt_00CA0000` (1280×1080) выглядит как полосатые цветные пятна с чёрным столбцом; правильно ли это (например, закодированные данные),
**не проверено**, видимое BMP делает `|v|`-тонмаппинг и показывает сырые каналы.

**Скорость:** на живом устройстве те же ~0,5 кадра/с (кадры 1460→1560 за ~210 с, с накладными расходами захвата), так что дело не в мёртвом GPU.
В RPCS3 предупреждение уходит примерно через 12 с. Если игра считает время кадрами, то при 0,5 кадра/с это ~12 минут: **гипотеза, не проверено**.

Новое: `D3D12_DRED=1` (хлебные крошки, ошибки страниц, имена ресурсов), отчёт `[D3D12-LOST]` один раз всегда, коды ошибок в `[RSXCAP]`.

### 2026-11-08 (вечер): профиль, шейдеры без кэша неудач (`patches/0008-fp-expand-helper-and-pso-negative-cache.patch`)

**Профиль до правки** (`logs/port-20261108-161035.log`, `PERF=1`): 0,37 кадра/с, `render_frame` 98%, половина в подготовке PSO, 1160 промахов на 20 кадров
(кэш на 256 записей при 85 разных ключах за кадр, так что не вытеснение).

**Ошибка 6 (исправлена): неудачные шейдеры не кэшировались.** `vp_get_fp_pso` при отказе возвращал NULL без записи в кэш, поэтому каждый неудачный
шейдер заново декомпилировался и компилировался при каждом обращении, дважды на отрисовку. Теперь неудача кэшируется (`pso = NULL`). Вытесняемый PSO
освобождается отложенно (та же опасность, что в ошибке 5). Заодно `_rsx_expand_input` теперь объявляется по факту вызова из тела шейдера: предварительный проход его
пропускал (лог `[FP] expand pre-scan missed…`). **Почему проход промахнулся, не выяснено.**

**Проверено запуском** (`-161747.log`, 300 с): промахов PSO 0, `render_frame` ~0,5 с на 20 кадров вместо ~52 с, 1,4–1,6 кадра/с. `-162313.log` (480 с, с сэмплером): 1,1–1,3 кадра/с.

**Экран:** кадр 1950 (`analysis/cap_prof2/frame_03`) — **предупреждение ушло**, маленький значок с сегментным кольцом, на вид индикатор загрузки. За экраном те же ~130 отрисовок (6 в бэкбуфер).

**Что теперь занимает время.** `guest` 96%. Сэмплы внутри exe (`rvas2.txt`, последние ~22 отчёта, 11 880 из 186 938, остальное ожидания/DLL): `spu_shufb` 37%, MLAA
(`edgepost_mlaa_*`) 20%, SPU-интерпретатор 15,5% (почти всё LS 0x37000–0x39C00: DSP-плагины Multistream, 11–13 млн шагов/с), прочий SPU-рантайм 15%, DMA/lockline 5%.

**Осталось:**
- 7 `PSO FAIL 0x80070057` (E_INVALIDARG), fp `0x09179141`, `0x09177F81`, `0x0916A001`, `0x09035A01`, `0x08F1FA01` (nrt=2), `0x00565E81`, `0x004FE881` (nrt=1): эти отрисовки не выполняются. Причина неизвестна.
- В `-162313.log` один SPU (`0x2002`) остановился: `HALT-ASSERT hlgti pc=0x00F00`, `stopped status=0x4 pc=0x00E60 FAULT`, r3=FFFFFFFF. SPURS не аварийный, остальные 4 работают. В `-161747.log` (300 с) этого не было. Не разобрано.

### 2026-11-08 (ночь): GETLLAR без блокировки и 20-минутный прогон (`patches/0009-spu-getllar-seqlock-halt-dump.patch`)

**Что за остановка SPU.** `HALT-ASSERT hlgti pc=0xF00` — проверка целостности в taskset PM (libsre). PM читает `GETLLAR` строку taskset в LS 0x2700 (running, ready, pending_ready, enabled, signalled,
waiting) и останавливает SPU, если `running & waiting`, `ready & pending_ready` или есть биты вне `enabled` (`port/spu/pm_taskset/spu_recomp.c:318-343`).

**Ошибка 7 (исправлена, связь с остановкой не доказана): быстрый GETLLAR не атомарен.** `spu_coh_reserve_fast` копирует 128 байт без блокировки; авторы считают, что при разорванной копии не пройдёт `PUTLLC`.
Но taskset PM проверяет копию **до** `PUTLLC`. Теперь seqlock: писатели под блокировкой меняют `g_lockline_seq`, читатель при его изменении перечитывает под блокировкой. На не-MSVC сборках всегда идёт блокировка.
`[spu-halt]` теперь печатает LS 0x2700–0x2760, чтобы при повторе было видно нарушенный инвариант.

**Проверено запуском** (`logs/port-20261108-170510.log`, 1200 с): `FAULT` 0, `HALT-ASSERT` 0, `D3D12-LOST` 0, `STUCK` 0, отрисовок 139 000+. Копия отличалась от перечитанной в 8+ случаях из 59 583+ перечитываний, все напечатанные —
`ea=0x10060600` (заголовок CellSpurs), не строка taskset. **Отсутствие остановки — слабое доказательство**: до правки она была в 1 из 12 прогонов.

**Скорость:** ~1,8–2,1 кадра/с на тяжёлом участке (было 1,4–1,6 / 1,1–1,3 с сэмплером; условия разные). `guest` ~10 с на 20 кадров. Начальные кадры идут на 6–8 fps вместо 144 fps в прошлых прогонах: **причина не выяснена** (возможно, это
цена seqlock). По стене времени до тяжёлого участка примерно те же ~45 с; счётчик кадров при этом меньше, поэтому сравнивать «кадр 1500» между прогонами нельзя.

**Экран:** кадры 1500–2400 (`analysis/cap_llar/`, 17:16–17:24) различаются, яркость текста меняется. На кадрах 1950 и 2400 две строки чёрным фоном с **частью букв**: «A» и «A  IN». Похоже на надпись загрузки с пропавшими глифами (**предположение**).
Снимок на 1190-й секунде: главный поток `lr=0x0067DF60`, `r3=0x1E` (тот же адрес, что «вариант В» 2026-11-06); один снимок.

### 2026-11-08 (ночь): ожидание на 0x67DF60 и кэш `/dev_hdd1`

**`lr=0x67DF60` — не семафор.** `func_007E89A4`: `while ([obj+0x2C] == 0) sys_timer_usleep(30)`; `r3=0x1E` — это 30 мкс (syscall 0x8D). Главный поток вызывает её для двух объектов `0x112F9E80+0x3084+i*0x30` перед `func_0067DC64`.
В `-175121.log` на снимках `[0x112FCF30]=0`, `[0x112FCF60]=1`, рядом указатели на `CellSpurs 0x10060600` и taskset'ы `0x11AC0580`/`0x11AC5E00` (в заметках они связаны с MLAA). 8 снимков из 9 на этом адресе, 1 на `0x15B314`: поток двигается. Похоже на покадровое
ожидание работы SPU (**предположение**).

**Кэш.** С заполненным кэшем чтение с диска кончается примерно на первой минуте (58,8 МБ, потом 0). Старый кэш перенесён в `vfs/hdd1_cache_backup_20261108` (оставлен). С пустым кэшем игра строит его сама через `chimp` + rename (`-175732.log`, `-180709.log`), как RPCS3.
**Мой вывод «стояло из-за старого кэша» оказался неверным:** новый кэш по списку файлов совпадает со старым (2830 файлов всего, расхождений в подкаталогах 0, 70 выборочных файлов побайтно равны). Кэш был исправен и полон.
Сборка заканчивается примерно на 80% лога `-180709.log` (последнее открытие файла — строка 138451 из 181400), дальше открытий нет. `FAULT` 0, `D3D12-LOST` 0, `HALT-ASSERT` 0 за 1200 с.
**Экран тот же:** кадр 2700 — те же «A / A IN». Скорость в конце 0,66–0,68 кадра/с (раньше ~2), причина не выяснена.

То есть после кэша загрузка не запрашивает файлов, а кадры продолжают идти. Файл `56c3842e` в RPCS3 открывается только на 10:18 (почти конец сессии), в порте с полным кэшем — в первую минуту: порядок отличается, причина неизвестна.

`PS3_SAMPLE_MAIN=1` роняет процесс на первом сэмпле (`-174608.log`, exit 139), причина не разобрана: не использовать.

### 2026-11-08 (ночь): потоки после кэша, `[CheckNetwork]`

**Потоки** (`-175121.log`, 9 снимков): `async_1..9`, все `fios *`, `JobBatchHandler`, `EvoJobWorkerThread` стоят в ожидании на всех снимках, работы им никто не выдаёт. Двигаются главный поток и `AmadeusDemoThread`.

**Сравнение с RPCS3 ограничено:** эталон снят с `Internet enabled: Connected`, RPCN и готовым сохранением; порт офлайн и без сохранения (`LOAD … new=1`). В RPCS3 после `async_7` идёт `cellNetCtlNetStartDialogLoadAsync`, callback в слоте 2, затем `sceNpEulaCheckEulaStatus`.
В порте ничего из этого нет (`Eula` 0 строк, `UnregisterCallback` 0, зарегистрирован только слот 0). Потоки, которых нет в порте: `SwapMusicWorkspaceThread` (8:11), `AvoidancePlanning`, `_cfg_evt_hndlr`, `libusbd_*`.

**Опрос сети.** Главный поток каждый кадр зовёт `cellNetCtlGetState` из `func_00370CA4` (lr `0x3715C8`, строки `[CheckNetwork] …` в EBOOT `0xF775C8…`; NSDU — по всей видимости, NetStartDialog). ps3recomp офлайн отвечает ошибкой `NOT_INITIALIZED` (подгонка под LBP), код при ошибке уходит на `loc_00371640` и повторяет на следующем кадре.
**Сделано:** `port/game_fixes.cpp` отвечает `CELL_OK, Disconnected` (`PS3_NETCTL_ERROR=1` — старое поведение). **Проверено запуском** (`-184111.log`, 390 с): переопределение срабатывает, но **загрузка не сдвинулась**: чтение те же 58,8 МБ, новых вызовов и потоков нет. На Disconnected код идёт на `loc_00371660` и остаётся на месте, пока байт `[[0x12E74AC]+0x30]` равен 0.
Этот байт пишут только `func_00370CA4`, `func_00370C40`, `func_0036F118`. Что `[CheckNetwork]` и есть блокер, **не доказано**. Правку оставил: она ближе к поведению консоли (предположение), вреда не показала.
В RPCS3 строк с вызовом `cellNetCtlGetState` нет (2 совпадения — импорты), сравнить частоту нельзя.

**Сетевой режим не включал:** `PS3_NET_ONLINE=1` даст реальные сетевые запросы, только с согласия пользователя.

**Автомат разобран: `[CheckNetwork]` — не блокер.** `func_00370CA4` — диспетчер по состоянию `[obj+0xC]` (13 значений, 0..0xC): 2 = `kNet_BringAdapterUp`, 3 = `NSDU_CHECK_NET`, 4 = проверка аккаунта PSN, 7 = «Player Signed into PSN», 0xA = проверка EULA, 0xB = «NOT SIGNED INTO PSN», 0xC = пусто (конечное). Функция отрабатывает за один вызов и возвращается, ничего не ждёт.
Запись `+0x30 = 1` делает `func_0036F118` при ненулевой настройке `…+0x78+0xE68`, сброс — «Resetting Net Check Mode», поэтому это, видимо, флаг «запрошена работа с сетью». Состояние 2 без адаптера и без флага — холостой ход.
Запуск (`-185616.log`, 300 с): на всех 16 напечатанных вызовах с #0 по #2048 `state=2 flag30=0`. Поэтому опрос `GetState` — фоновый, и моя догадка о нём как о причине была **неверной**. Переопределение ответа оставлено, на загрузку не влияет; лог в нём теперь редкий.
Отсутствие EULA/NetStart в порте — следствие офлайна (флаг не поднят), а не поломка (предположение).

Стек главного потока при ожидании: `func_007E89A4` ← `func_0064C940/70` ← `func_005C4F70/FD0` ← `func_005C869C` ← `func_005CAD38` ← `func_005C87B8` ← `func_0067F080` ← `func_00688954` ← `func_005C65EC` ← `func_0067EC18` ← `func_005CAC58` ← `func_0078B1D8` ← `func_0076E2D8` ← `func_00786B18` ← `func_00775FA0` ← `func_004DE1F0` ← `func_005C5FAC` ← `func_007F6998`.

### 2026-11-08 (ночь): собственный отладочный лог игры

Функции стека главного потока строк не содержат (это обход дерева кадра), на `bootToFrontend`/`BootState`/`LoadingScreen` прямых ссылок из кода нет (видимо, таблицы данных).

**Новый инструмент: `PS3_GAME_LOG=1`.** `func_00AE0E68` и `func_00B88374` в retail пустые, но получают строку формата (4047 и 166 прямых вызовов). `tools/patch_gamelog.py` вставляет в **сгенерированный** код вызов `port_game_log` (`port/game_fixes.cpp`). **После каждого нового лифта PPU скрипт надо запускать заново.** Печать: целые и строки, плавающие как `<float>`, аргументы со стека `<stack>`; каждый формат не больше 12 раз, поэтому покадровые сообщения после 12-го не видны.

Прогон `logs/port-20261108-190929.log` (240 с, полный кэш): 186 сообщений, все главные этапы инициализации сети и логики гонки (`Creating …Action`, `…Logic::ExitLogic`, список треков в потоке 16). `FAULT` 0, `D3D12-LOST` 0.
- `Unable to acquire main user account region information, error code: 0x8002aa0c` — это `SCE_NP_ERROR_OFFLINE` (`repo/libs/network/sceNp.h:30`), для офлайна ожидаемо.
- **Последнее сообщение — `LogicFullyActive [OTGConnectionLogic]`** (lr `0x36B08C`, строка 4315 из 38 476). Дальше новых сообщений нет. OTG, судя по строкам (`OTG NOT CONNECTED`, `Connecting to OTG...`, `SubSystemConfig/Online/OTGC/...`), — онлайн-сервис игры. Держит ли он загрузку, **не проверено**: молчание может значить и то, что дальше игра просто не пишет в лог.

**`OTGConnectionLogic` — не блокер.** `func_0036AF78` (вызов из `0x36D21C`) — менеджер компонентов: обходит список и один раз печатает `LogicFullyActive` при активации, ничего не ждёт. Строка `OTGConnectionLogic` используется в `func_00C48864`.

**Гипотеза «экран ждёт кнопку» проверена и не подтвердилась.** Основания были: `AsyncRenderThread` завершается после `async_9`, рабочие потоки простаивают, файлы не читаются, в EBOOT есть `PressStart`, `kTitleScreenLogo`, а все прогоны автоматические. Прогон `logs/port-20261108-191639.log` (480 с), `PAD_SCRIPT` с нажатиями START на 120/240/360 с и CROSS на 180/300/420 с:
все 6 нажатий поданы, после первого нажатия **ни одного нового сообщения игры, потока и открытия файла**; чтение 58,86 МБ (+11 КБ за весь прогон). Ограничения: проверены только START и CROSS, и то, что игра обработала данные пада, не проверено (нет трассировки чтения кнопок гостем).

Экран циклически меняется: хэши backbuffer `f3a3ad97`, `eedf3bec`, `d0945892`, `25611f15`; номер кадра с содержимым между прогонами не совпадает.

**Кто создаёт `SwapMusicWorkspaceThread`.** Дескриптор `0x12873F0`, имя `0xF58EF0`; создаёт `func_000AED50` (строка `musiclayer`). Прямо её зовут только `func_00B0AFC8` (`gui/gui.wacd`, `start`, музыка интерфейса) и `func_00B0B228` (`prologue_veteran.wacd`, музыка пролога). Обе — первые указатели в записях таблицы состояний по 3 указателя на запись, начало `0xFD16A8`, индексы 20 и 21; вторые указатели — обновление (`0xB0B054`, `0xB0B1E0`), видимо, вход/обновление/выход (**предположение**).
В эталоне поток создаётся на 8:11, в том же месте `async_3` делает `cellSaveDataUserAutoLoad(BCUS98242-GSTS)`; в порте этот вызов идёт раньше, на старте (`-175121.log`:1451), так что порядок разный. Кто читает таблицу, пока не найдено: прямых ссылок на начало нет.

**Поправка: «таблица состояний 0xFD16A8» — моя ошибка.** Это таблица виртуальных методов на 79 слотов, начало `0xFD16B4`, класс звукового менеджера; конструктор `func_00B06558` (цепочка вызова `func_000AF858` ← `func_002B2838` ← `func_002ADF88`). `func_00B0AFC8` — слот 57 (смещение `0xE4`), `func_00B0B228` — слот 60 (`0xF0`).
`PS3_MUSIC_TRACE=1` (`port/game_fixes.cpp`, через `ppu_register_function`): в `logs/port-20261108-194453.log` (240 с) **ни один из двух методов не вызван**, т. е. до запуска музыки интерфейса игра не доходит. Вызовов со смещением `0xE4` в коде 12, из них в этом же классе — `func_00B09B38` (слот 7) и `func_00B0B718` (слот 38).

**Перехват всех методов класса** (`PS3_MUSIC_TRACE=1`, 74 уникальных метода, `port/music_trace.inc`; печатаются первые 6 вызовов каждого и потом каждый 4096-й). `logs/port-20261108-195045.log` (240 с): перехват работает, вызваны слоты 2, 7, 11, 14, 15, 20, 68, 70, 71, 73, 77, 78; слоты 57 и 60 — **нет**. Менеджер звука: `this=0x18F5BE50`, доступ `[[0x12ECE5C]+0x298]+0xB4`.
Слот 7 вызван один раз, с аргументом 0, из самого класса (`lr=0xB07DDC`), при инициализации. Слоты 14, 15, 20, 68 вызываются и у других объектов (общие методы базового класса), это не менеджер.
Статический поиск по `+0x298` → `+0xB4` → метод (образец, 15 строк окно, мог что-то пропустить): слот 57 зовут `func_001BA2B0`, `func_002A9648` (ret `0x2A969C`), `func_005392DC` (ret `0x5393E8`); слот 7 — `func_001D16C4`, `func_001EFC20`, `func_004D320C`, `func_004D6620`; слот 60 — `func_002A9230`, `func_002A9588`, `func_004FB2B8`.
`func_002A29C0` (из неё вызван слот 73) содержит `switch` по `func_001A69FC([r31+0x14])`, значения 0..10; вызов `0xE4` на `0x2A2AAC` там идёт к **другому** объекту (`+0xD8`), не к менеджеру звука.

**Трассировка входа в функции (`PS3_FUNC_TRACE=1`).** `tools/patch_gamelog.py` теперь вставляет `port_func_trace` в начало 10 функций в сгенерированном коде (после нового лифта запускать заново). `logs/port-20261108-195938.log` (240 с):
- все три функции, зовущие слот 57 (`func_001BA2B0`, `func_002A9648`, `func_005392DC`), и все три, зовущие слот 60: **0 входов**. Игра до этих мест не доходит;
- из функций слота 7 выполнялись `func_004D320C` (2 входа, lr `0x4D5EBC`) и `func_001EFC20` (не меньше 8, lr `0x1E0EFC`, логи 4875–5039 на старте), но слот 7 за прогон вызван только один раз при инициализации, т. е. их ветка с вызовом не сработала; `func_001D16C4`, `func_004D6620`: 0 входов.
Прямой вызов есть только у `func_005392DC`: `func_001B1938` ← `func_001B2660` (та вызывается косвенно). `func_001BA2B0` и `func_002A9648` прямых вызовов не имеют.

**Подъём выше: классы.** `func_001B2660` и `func_001BA2B0` — слоты 6 и 16 таблицы `0xFDDBC0` (30 методов, конструктор `func_001ACD50` пишет `0xFDDBB8`, строки `Custom Game Mode`, `Game/Finished`) — класс режима гонки. `func_002A9648` — слот 3 таблицы `0x1003E80` (9 методов, строка `loading`, создаётся из `func_00C42CA0`, та — слот 2 маленькой таблицы `0x10026FC`).
`logs/port-20261109-183353.log` (240 с): конструкторы и методы обоих классов (`func_001ACD50`, `func_001AD900`, `func_00C42CA0`, `func_002A94CC/951C/96C4/97B8/9824`) — **0 входов**.

**Вывод о якоре (поправка, вывод косвенный):** `SwapMusicWorkspaceThread` на 8:11 в эталоне, вероятно, относится ко входу в гонку, а не к меню: через 8 с появляется `AvoidancePlanning`, а все три функции слота 57 лежат в классах гонки/загрузки. Значит, по нему нельзя судить о том, где порт стоит: подъём по этой цепочке искал следующий этап **после** меню. В эталоне меню, видимо, идёт между 7:15 и 8:11.

**Внеэкранные цели** (`analysis/cap_pad/frame_04`, кадр 1900, 134 отрисовки): `rt_0058CE00` (1280×720, float) — ряд одинаковых полосатых столбиков, `rt_01740000` — почти чёрный с одной шумной линией. Узнаваемой сцены нет. Нельзя сказать, что игра уже в меню, но и рисуется она явно не как задумано (**предположение**). Последние операции кадра: 10 отрисовок в backbuffer (FP `0x003FE341`, `0x00367201`, `0x003A7441`, `0x003E7981`).

**Кнопки: гость их получает.** `PAD_TRACE=1` (`repo/libs/input/cellPad.c`, правка пока не в патче), `logs/port-20261109-184118.log` (330 с): `cellPadGetData` зовёт поток `FpPadThread` (lr `0x7F92A8`, порты 0–5), за прогон ~30 000 чтений, из них непустых ~20 000. Пакеты с нажатием приходят с `len=28`, `d1=0x08` (START) и `d2=0x40` (CROSS). Версия «вход сломан на уровне HLE» **снята**. Доходит ли нажатие от `FpPadThread` до логики меню, не проверено.
Реакции не видно: после нажатий 0 новых сообщений игры. Backbuffer сменился с `f3a3ad` (предупреждение) на `eedf3b` (чёрный) между F1545 и F1560, сразу после CROSS на 210 с, но та же смена происходит и в прогонах без нажатий (`cap_net`, `cap_fresh2`), так что нажатию её приписать нельзя.

**Находка в отрисовке.** В кадре (`analysis/cap_pad/frame_04`) последняя внеэкранная отрисовка перед backbuffer — `op123`, полноэкранный проход `fp=0x00565E81 vs=34` в цель `0x02C80000`. Этот же шейдер есть в списке 7 отказов (`PSO FAIL fp=0x00565E81 … vs=34 nrt=1 rtfmt=28`, E_INVALIDARG). То есть как минимум один проход финальной сборки кадра не выполняется. Связь с чёрным экраном **предположение**.

**Следующий шаг:** (1) причина `PSO FAIL 0x80070057` (debug layer `D3D12_DBG=1`, сообщения InfoQueue при создании PSO; сравнить VS-выход и PS-вход); (2) пропавшие буквы, полосы в `rt_0058CE00`; (3) доходят ли нажатия от `FpPadThread` до логики; (4) поведение без сохранения (`new=1`); (5) 0,66 кадра/с, `spu_shufb`; (6) `[obj+0x2700]`, `AmadeusDemoThread`.

### 2026-11-10: 16 текстурных юнитов, debug layer (записано 2026-11-11 задним числом)

- **Ошибка 8 (исправлена): всего 4 текстурных юнита в D3D12-бэкенде.** Шейдеры игры берут сэмплеры до s11; все 7 `PSO FAIL 0x80070057` были из-за этого.
  `repo/libs/video/rsx_d3d12_backend.c`: `VP_TEX_UNITS 16` (корневая сигнатура, окна SRV на отрисовку, сэмплеры), пул внеэкранных целей 16 → 64, depth 6 → 16,
  `DRAW_SRV_BASE 128`; операция, для которой цель не нашлась, отбрасывается, а не рисуется в backbuffer. Патч `patches/0010-d3d12-16-texture-units-bigger-target-pools.patch`.
  **Проверено запуском** (`logs/port-20261110-104940.log`, 300 с): `PSO FAIL` 0 (было 7). Картинка на экране от этого не изменилась.
  Ограничение: `rsx_texscale[]` в константном буфере b1 по-прежнему на 4 юнита (юниты 4+ с ненормализованными координатами получат масштаб 1).
- `PAD_TRACE=1` вынесен в `patches/0011-pad-trace.patch`.
- Debug layer (`D3D12_DBG=1 D3D12_IQ=1`, тот же лог): сообщение 613 — PSO формата `R16G16B16A16_FLOAT` рисует в цели `R8G8B8A8` (`rt 0x021E0000 320x270 rsxfmt 08`,
  `0x03810000`, `0x04310000 320x180`, `0x043F1000 160x90`, `0x04429400 80x45`, `0x00589700 40x22`). Разбор ниже (ошибка 10).

### 2026-11-11: пропавшие буквы — кэш текстур; на экране меню «PLAY / PLAY IN 3D»

- **Прогон с `TEX_NOCACHE=1` от 10 ноября не был потерян** (`logs/port-20261110-105527.log`, 240 с, `analysis/cap_nocache/`, 3 кадра). На кадрах 1620–1860 чётко читается
  **«PLAY / PLAY IN 3D»** вместо «A / A IN». Гипотеза про кэш текстур **подтверждена**.
- Значит, игра не «висит после предупреждения»: это **меню выбора 2D/3D, и оно ждёт игрока**. Простой рабочих потоков и отсутствие чтения файлов — нормальное состояние меню.
  Прежние проверки кнопок (START/CROSS по 40 опросов ≈ 0,4 с при 2–3 кадрах/с) реакции не дали; почему — см. ниже.
- **Ошибка 9 (исправлена): редкая контрольная сумма текстур.** `tex_csum` брала 1024 отсчёта с шагом `nbytes/1024`. Для атласа глифов 2048×1024 B8 шаг равен ровно одной строке (2048 байт),
  то есть читался только столбец 0, и дорисованные в атлас глифы не замечались: текстура на GPU оставалась старой. Теперь хэш по всем байтам (4 потока по 64 бита);
  `TEX_CSUM_SPARSE=1` — старое поведение. Патч `patches/0012-full-texture-checksum.patch`.
- **Проверено запуском** (`logs/port-20261111-183300.log`, 420 с, без `TEX_NOCACHE`, `analysis/cap_csum/`): кадр 1500 (~150 с) — «PLAY / PLAY IN 3D», побайтно тот же backbuffer,
  что в прогоне с `TEX_NOCACHE` (`25c5ddde`); кадр 1700 — экран «This game saves data automatically…» с полным текстом. Хэширование дешёвое: `tex` 0,01–0,02 с на 20 кадров.
- **CROSS в меню работает.** `PAD_SCRIPT=220:0x4000,…,225:0x4000` (шесть нажатий подряд, ~6 с удержания): сразу после этого игра открывает
  `/dev_bdvd/PS3_GAME/USRDIR/ui/video/evologo_60.avi`, создаёт ~12 потоков, зовёт `cellVdec Open(codecType=5)` (DivX), `StartSeq`, `SetFrameRate(132)`, 7 × `DecodeAu`.
  Кадры идут на **144 кадра/с** с 2 операциями, backbuffer **белый** (`244b8bed`), и так до конца прогона (~200 с): ролик не заканчивается.
  Почему раньше нажатия «не работали»: вероятно, они приходились на экраны предупреждения/автосохранения, а не на меню, либо 0,4 с удержания мало при 2–3 кадрах/с (**не проверено**, какой из двух).
- **Новый блокер: видеоролик DivX.** В ps3recomp `cellVdec` для DivX — заглушка (AUDONE + PICOUT с чёрной картинкой, `repo/libs/codec/cellVdec.c:448`, `cellVdecDivx.c` — «Stub»).
  В эталоне RPCS3 в этот момент грузится `libdivxdec.sprx` (3:11 в baseline). Новый нерешённый NID на этом пути: `0x8B5551A4`.
- **Ошибка 10 (исправлена): пул внеэкранных целей с ключом «только смещение».** Игра в одном кадре использует одно смещение VRAM под разные поверхности
  (`0x021E0000`: цель 320×270 в op33 и 1280×1080 в op45–55; у многих смещений есть и RGBA8, и float-вариант). Бэкенд держал один ресурс на смещение: предпроход пересоздавал его
  на каждую такую отрисовку (с `wait_for_gpu`), побеждал последний размер/формат, остальные проходы рисовали PSO не того формата в цель не того размера (сообщение 613).
  Теперь ключ — (смещение, размер, формат); текстура ищет цель по смещению + размеру + классу формата (float/не float), иначе самую свежую по смещению; вытесняется самая давняя.
  `RT_KEY_OFFSET=1` — старое поведение. Файлы захвата теперь `rt_<off>_<w>x<h>_f<dxgi>.bmp`. Патч `patches/0014-d3d12-offscreen-targets-keyed-by-offset-size-format.patch`.
  **Проверено запуском** (`logs/port-20261111-184108.log`, 300 с, debug layer): сообщений 613 — **0** (было десятки), `D3D12-LOST` 0, `PSO FAIL` 0. Экраны выглядят так же
  (они рисуются прямо в backbuffer). Правильность самих внеэкранных картинок **не проверена** — сравнивать пока не с чем.
  Остались предупреждения debug layer: 679 (шейдер пишет в RT 1–3, которых в PSO нет), 820/821 (значение очистки не как при создании) — безвредные.
- **Порядок экранов** (по кадрам трёх прогонов): предупреждение об эпилепсии → «PLAY / PLAY IN 3D» → после CROSS «This game saves data automatically…» → ролик `evologo`.
  Момент появления меню по стене времени плавает (150–200+ с), поэтому одиночное нажатие по времени ненадёжно: в `-184721.log` нажатия на 200–205 с не сработали, ролик за 480 с не начался.
  Для автопрогонов нажимать CROSS с периодом (например, каждые 12 с).
- `cellAdecOpenEx` (NID `0x8B5551A4`) в ps3recomp нет — добавлен в `port/game_fixes.cpp` (передаёт в `cellAdecOpen`, HLE-декодер отдаёт тишину). Гипотеза: ролик стоял из-за него
  (плеер ждёт звук). Результат проверки — ниже.
- **Меню само не уходит** (`logs/port-20261111-190556.log`, 570 с, `analysis/cap_video3/`: из-за моей ошибки в сценарии — в Git Bash нет `bc` — все нажатия ушли в 0-ю секунду, то есть
  прогон фактически без нажатий): «PLAY / PLAY IN 3D» стоит с ~200-й секунды до конца, выбранный пункт пульсирует (поэтому хэши кадров разные), время от времени на экран
  возвращается надпись про автосохранение. Короткие нажатия (40 опросов ≈ 0,4 с) при 1,5–3 кадрах/с игра обычно пропускает: в `-185558.log` 34 таких нажатия ничего не дали.
  **Рабочий сценарий:** удержание ~3 с (9 событий `PAD_SCRIPT` с шагом 0,35 с) каждые 10 с; генерировать Python'ом, не `bc`.
- **Ролики проходят** (`logs/port-20261111-191606.log`, 570 с, с `cellAdecOpenEx` и удержанием CROSS на 180–430 с). **Проверено запуском:**
  `cellAdec Open(codecType=2)` (AC-3) → `evologo_60.avi`: 382 видео-AU и 399 аудио-AU, `EndSeq`/`Close` → `maintitles_60.avi` → `Close` (второй, видимо, пропущен нажатием) →
  игра открывает `/dev_hdd1/4/4b8d2298`, пишет `LogicFullyActive [AnnouncementDisplayLogic]` и рисует тяжёлые кадры на ~2,7 кадра/с до конца прогона. `FAULT` 0, `D3D12-LOST` 0.
  Гипотеза про `cellAdecOpenEx` **подтверждена**: без него ролик стоял на 7 AU (`-183300.log`).
  Картинки в роликах нет: `GetPicture` 0 вызовов, экран белый (`244b8bed`); звук — тишина. DivX (MPEG-4 ASP, 1280×720) и AC-3 в ps3recomp не декодируются.
  Настоящее видео: в ps3recomp есть путь через FFmpeg (`PS3RECOMP_FFMPEG_RUNTIME`, только H.264/MPEG-2; для DivX нужно добавить `AV_CODEC_ID_MPEG4`), нужны заголовки и
  `avcodec-*.dll` той же мажорной версии. На машине FFmpeg 8.1.1 только в виде exe (`C:\ffmpeg\bin`), без DLL и заголовков. Скачивание — **только с согласия пользователя**.
- **После роликов — экран объявления «MotorStorm Apocalypse After Party»** (`logs/port-20261111-192620.log`, 660 с, `analysis/cap_after_video/`, скриншот
  `analysis/after_party_screen.png`; меню — `analysis/play_menu_screen.png`). **Проверено запуском:** кадр 6500 — цветной логотип и три строки текста, кадр 7000 — добавилась
  подсказка «✕ CONTINUE». Игра во фронтенде и ждёт CROSS. ~2,6 кадра/с, 130–139 операций на кадр, фон чёрный (должен ли за текстом быть 3D-фон — **неизвестно**).
  `FAULT` 0, `D3D12-LOST` 0, `HALT-ASSERT` 0. Хронология по стене: меню ~200 с, ролики ~310–350 с (на 144 кадра/с), объявление с ~430 с.
- **Патчи.** Старый `0005-spu-interp-irq-and-diag.patch` не накладывался на чистый ps3recomp (серия не воспроизводила дерево). Он перенесён в `patches/superseded/`, вместо него
  `0013-spu-interp-irq-and-diag-regenerated.patch` (накладывается последним). Проверено в отдельной рабочей копии: серия `00*.patch` по порядку даёт ровно `ALL-repo-changes.patch`.

### 2026-11-11 (вечер): скорость — 2,7 → ~7 кадров/с в меню; настоящий тормоз — MLAA и вывод кадра

Замеры в меню «PLAY / PLAY IN 3D» (те же ~130 операций на кадр), без захвата кадров.

- **Ошибка 11 (исправлена): переведённый SPU-код собирался без `-msse4.1`.** Быстрые `spu_shufb`/`spu_selb` (pshufb/blendv) в `spu_helpers.h` включаются по `__SSE4_1__`.
  Runtime получает флаг от ps3recomp, а SPU-исходники порта (`port/CMakeLists.txt`, `SPU_SOURCES`) собирались только с `-O2`, и там работал побайтный цикл
  (37 % сэмплов в профиле 2026-11-08). Теперь `-O2;-msse4.1`. Это правка порта, не ps3recomp.
- **VSync.** После этого `guest` упал до ~0,03 с на кадр, а кадр упёрся в ожидание GPU в начале `render_frame` (~0,2 с, новые поля PERF `gpuwait`/`rec`,
  `patches/0015-perf-gpuwait-and-record-time.patch`). Загрузка GPU при этом 1 % (`nvidia-smi`). С `D3D12_VSYNC=0` ожидание 0. `tools/run_port.sh` теперь ставит
  `D3D12_VSYNC=0` по умолчанию. Почему `Present(1)` ждёт так долго (окно перекрыто при автопрогоне? Optimus?) — **не выяснено**; при игре руками проверить с `D3D12_VSYNC=1`.
- **Проверено запуском:** 2,6–3,0 кадра/с (`-183300.log`) → 4,2–5,0 с SSE4.1 (`-195626.log`) → 6,5–8,5 без VSync (`-200635.log`: 8,4; `-203609.log`: 6,5–7,5; разброс между прогонами есть).
  `FAULT` 0, `D3D12-LOST` 0 во всех.
- **AVX2 не помог** (`PORT_AVX2`, опция в `port/CMakeLists.txt`, по умолчанию OFF, `build_port.bat` передаёт `-DPORT_AVX2=OFF`): с `-mavx2 -mfma` включаются 4-полосные
  fa/fm/fma, но данные MLAA почти всегда уходят на запасной путь (`spu__x8_fma_fix` 14 %, `spu__x8_fm_fix` 4 %), итог 5,3 кадра/с против 6,9 (оба с сэмплером). Вероятная причина —
  MLAA считает мусор (см. ниже), **не проверено**.
- **Что держит кадр сейчас** (`PS3_FTRACE=200:10:<файл>` + `repo/tools/ftrace_report.py`, медиана кадра 137 мс): главный поток 113 мс из 137 ждёт в `usleep@0x67DF60`
  (`func_007E89A4`, ожидание задач MLAA), RSX занят ~9 мс, остальные потоки спят. Профиль (последняя треть, только код exe): `edgepost_mlaa_spu_func_00008660` 37–40 %,
  `spu__fma_any` 13 %, `spu_step` (интерпретатор DSP Multistream, отдельный SPU, кадр не держит) 10–12 %, `spu_lifted_lookup*` 8 %. В среднем занято меньше одного ядра:
  MLAA идёт на одном SPU последовательно, ~50–65 мс процессора на кадр.
- **Как игра выводит кадр (`NV3089_DBG=40`, `-203037.log`) — это же причина чёрного фона:**
  1. сцена собирается в цель `0x02C80000` (1280×1080, op122, FP `0x00565E81`);
  2. четыре блита NV3089 копируют её из VRAM в основную память `0x1200A000` (плитки 1024×1024, 256×1024, 1024×56, 256×56, pitch 5120);
  3. SPU делает MLAA на месте в `0x1200A000`;
  4. блит 1280×720 из `0x1200A000` в буферы экрана `0xC0590000` / `0xC0914000`; поверх рисуется интерфейс.
  У нас содержимое цели есть только на GPU, в гостевую память оно не попадает, поэтому шаги 2–4 гоняют нули/мусор: фон чёрный, а MLAA тратит ~60 мс на кадр впустую.
  (Это то, для чего RPCS3 нужен `Write Color Buffers`.) В ps3recomp есть заготовки: `GCM_COMPOSITE_RT`, `GCM_GUEST_FB`, `rsx_live_draw_*` — не пробовал.

### 2026-11-11 (ночь): вывод сцены прямо с GPU (`patches/0016-scene-blit-from-gpu-target.patch`) — сделан; пропуск MLAA — НЕ сделан

Решение пользователя: цель — 144 кадра/с, поэтому идём путём (б): кадр берём с GPU, MLAA на SPU пропускаем.

- **Сделано: `GCM_RT_BLIT_COMPOSITE=1`** (в `tools/run_port.sh` включено по умолчанию). `cellGcmSys.c` (`nv3089_blit`): блит из GPU-цели в основную память запоминает
  «адрес → цель» (лог `[NV3089] main 0x1200A000 now aliases offscreen target 0x02C80000`); последующий блит ≥1280×720 из этого адреса в локальную память вызывает
  `rsx_d3d12_blit_rt_to_display`. Бэкенд на этом месте порядка операций рисует цель полноэкранным треугольником (линейный фильтр, 1280×1080 → 1280×720) в backbuffer, потом идёт интерфейс.
  `GCM_RT_BLIT_DEBUG=1` — отбросить операции в backbuffer после блита (видна только сцена).
- **Проверено запуском** (`logs/port-20261111-204636.log`, `-205147.log`, `analysis/cap_sblit/`, `analysis/cap_sblit_dbg/`): блит срабатывает каждый кадр; в отладочном режиме
  backbuffer совпадает с содержимым цели `0x02C80000` (яркое пятно со зданием у правого края). В обычном режиме меню остаётся чёрным: интерфейс рисуется поверх
  (вероятно, игра сама затемняет фон на этом экране — **не проверено**). `FAULT` 0, `D3D12-LOST` 0, скорость та же (~7 кадров/с).
- **Сцена за меню рисуется, но итог почти пустой.** `rt_00CA0000` (1280×1080) — полноценный буфер сцены с силуэтом зданий (упакованные данные, полосы), `rt_01C90000` тоже заполнен,
  а в итоговой цели `0x02C80000` ненулевых ~2 % (одно яркое пятно). Либо игра сама держит фон тёмным на этих экранах, либо ломается проход освещения/сборки. **Не разобрано**;
  проверять на экране, где фон точно должен быть виден (главное меню).
- **Не сделано: пропуск MLAA.** Главный поток по-прежнему ждёт задачу (`func_007E89A4`, флаг `[obj+0x2C]`). Начал искать место: в `recompiled/ppu_recomp_032.cpp` в `+0x2C` пишут
  `func_007E8854` (строка 2765), `func_007E8A84` (инициализация, 3099) и `func_007E99A0` (3987, 4001) — кандидаты на «запуск» и «готово». Идея: перехватить функцию запуска
  (как `PS3_GATE_TRACE` в `port/game_fixes.cpp`), чтобы она сразу ставила флаг готовности и не будила SPU. Переведённый SPU-образ MLAA порезан на фрагменты (`SPU_DIRECT`), по нему
  граф вызовов не строится.

**ГДЕ СТОИМ (2026-11-11, вечер):** игра проходит предупреждение → меню 2D/3D → два ролика (без картинки и звука) → экран «After Party» с «✕ CONTINUE». Дальше не нажимали.

**Следующие шаги:**
1. Нажать CROSS на «After Party» и идти дальше к главному меню (сценарий: удержания до ~430 с, пауза, снова удержания; захват `RSX_CAP` с ~6500-го кадра). Каждый следующий
   экран, скорее всего, даст новые нерешённые NID — смотреть `unresolved NID` после 126 000-й строки лога.
2. **Вывод кадра и MLAA (и скорость, и картинка)** — см. раздел «скорость» выше. Два пути:
   (а) честный: при блите NV3089 из цели, которая живёт на GPU, читать её обратно в гостевую память (readback 1280×1080 на кадр), тогда MLAA и финальный блит заработают как на консоли,
   но кадр останется ~60 мс MLAA на одном SPU;
   (б) обход: финальный блит 1280×720 брать прямо из GPU-цели `0x02C80000` (композит на GPU), а задачу MLAA завершать сразу. Быстрее и проще, но без сглаживания и это правка под игру.
   Моё мнение: сначала (б) как переключатель, чтобы увидеть сцену и получить скорость; (а) — потом, если понадобится точность. Сначала попробовать готовый `GCM_COMPOSITE_RT`.
3. Видео в роликах: решить с пользователем про FFmpeg (DLL + заголовки) и добавить DivX → `AV_CODEC_ID_MPEG4`; AC-3 — аналогично через `cellAdec`. Не блокер.
4. Проверить внеэкранные цели после ошибки 10 (что в `rt_*_f10/f28`), нужен ли 3D-фон во фронтенде; `rsx_texscale` на 16 юнитов.
5. Старое: `[obj+0x2700]`, `AmadeusDemoThread`, треск звука, начальные кадры 6–8 fps в части прогонов.

Следующий этап — SPU. Без него игра не распакует ни одного ассета.
000. **Актуально сейчас:** (1) прогнать дольше и посмотреть, что идёт после предупреждения
   (заставки, меню); (2) скорость: профиль `PS3_SAMPLE` и `SPU_INTERP_STATS` на этом участке;
   (3) заглушки `cellHttpClientSet*`, `PollConnections`, Mediator, Usbd, `cellRtcGetDosTime`;
   (4) треск звука.
00. ~~Шрифты~~ (сделано, LLE). Предлагаемый путь — LLE `libfont.sprx`, `libfontFT.sprx` и `libfreetype.sprx`
   тем же конвейером, что libsre (`prx_load_module`, экспорты в приоритете над HLE). Пользователь
   расшифровывает копии в `firmware/`. Альтернатива — переписать HLE cellFont (FreeType init,
   правильные сигнатуры, перевод путей, рендер глифов), но это большой объём.
   Затем: `sys_mempool_*` (HLE по образцу RPCS3 или LLE liblv2), заглушки Mediator/Usbd/HttpClientSet*.
0. ~~Актуально после LLE~~ (сделано): понять, что именно ядро SPURS выполняет в интерпретаторе
   (LS 0x0C00–0x1000, 0x8000–0x8800). Снять дамп LS и сверить с Wws PM (EBOOT 0x123B100) и
   с raw-блобами libsre, затем перевести эти policy-модули (у ps3recomp есть пример
   `--extra-funcs pm_wwsjob=0xA2C` и `taskset_policy_bytes.c` в шаблоне). Проверить, кто должен
   писать в очередь событий q=2 (edgezlib через LFQueue?) и доходит ли туда событие.
1. ~~edgezlib~~ сделано.
2. Развилка для LFQueue ANY2ANY (и дальше для всего SPURS):
   - (a) реверс протокола ANY2ANY по SPU-стороне edgezlib → свой HLE push в `port/`. Объём
     неизвестен, ошибки будут тихими: данные потеряются или задача будет висеть;
   - (b) **LLE libsre** (`docs/FIRMWARE_LLE.md`, сделано на Yakuza: Dead Souls и inFamous). Нужен
     **расшифрованный** `libsre.prx`. Расшифровку ps3recomp оставляет пользователю: например,
     RPCS3 → Utilities → Decrypt PS3 Binaries над **копией** `dev_flash/sys/external/libsre.sprx`,
     не над оригиналом в `C:\rpcs3`. Это также закрывает Taskset2/Task2 и Wws_Job.
3. Извлечь и перевести `Wws_Job` policy module (0x123B100, 12 160 байт). В build_spu_workloads есть
   пример `--extra-funcs pm_wwsjob=0xA2C`: похоже, этот PM ps3recomp уже встречал. Затем ловить
   job-модули через `SPU_DUMP_MISS`.

### 2026-11-21: three small ps3recomp fixes from the EU (BCES00484 1.06) port (patches 0017-0019)

- `0017` vdec: `CELL_VDEC_CODEC_TYPE_DIVX` -> `AV_CODEC_ID_MPEG4` in the FFmpeg path (the intro movies are DivX / MPEG-4 ASP; this is the line noted above as missing). Needs `PS3RECOMP_FFMPEG_RUNTIME` and FFmpeg shared DLLs (tested with FFmpeg 8.1 LGPL shared). **Built and running in the EU port, but the picture is not verified there** (that port skips the intros), so it needs a run here with the intros on.
- `0018` cellFs: `cellFsRename` deletes an existing target first. PS3/POSIX rename replaces it, Windows `rename()` fails, so saving over an existing file failed. Not run-tested in this port.
- `0019` cellFont: glyph scale by the em (`stbtt_ScaleForMappingEmToPixels`) like the real FreeType-based library, instead of ascent-descent (`ScaleForPixelHeight`), which drew text smaller than the game lays it out. In the EU port since 2026-11-04 (health warning and menus draw with the game's own fonts); no A/B comparison in this port yet.
- Checked: the series applies on a clean ps3recomp `4e8f3ca`, and `ALL-repo-changes.patch` (0017-0019 appended) applies and gives the same three files.

### 2026-11-23: four rendering fixes from the EU (BCES00484 1.06) port (patches 0020-0023)

- `0020` cellVideoOut: `GetResolutionAvailability` answers 0 for the 3D ids (>= 0x80), as a 2D TV / RPCS3's default does. It said yes to 129 and 136-138, so the game took its 3D/SimulView path (a SimulView notice when entering Wreckreation). Tested in the EU port: the notice is gone and menus/races still boot. `PS3_STEREO3D=1` restores the old answer.
- `0021` d3d12: the guest scissor (`NV4097_SET_SCISSOR_*`, already parsed into `rsx_state`) is recorded per draw and intersected with the viewport rect. Before, only the viewport clipped. Tested in the EU port: Quick Race track select shows exactly 3 cards and clips them while scrolling; the event-name splash stays inside its panel. `SCISSOR_OFF=1` = old.
- `0022` d3d12: `VP_TEX_SLOTS` 192 -> 2048, plus an offset hint table so a bind does not scan all slots. A race frame on The Rock binds ~275 distinct textures, every slot is `used` for the frame, so the draws recorded last (HUD swooshes and digits, boost gauge, some riders/liveries) got a NULL SRV and drew black or not at all. Tested in the EU port: rider lit, HUD complete in races.
- `0023` d3d12: a triangle strip split into several `DRAW_ARRAYS`/`DRAW_INDEX_ARRAY` packets (<= 256 each) inside one BEGIN/END is joined again: a packet that continues the previous one re-reads its last 2 vertices and keeps the winding parity. Each packet was its own strip, so 2 triangles were lost at every boundary (small holes in rider meshes). **Built, not yet confirmed in a run.** `STRIP_NOCONT=1` = old.
- Checked: the four patches were made on a clean `4e8f3ca` + the old `ALL-repo-changes.patch`; the new `ALL-repo-changes.patch` (0020-0023 appended) applies on `4e8f3ca`, and `rsx_d3d12_backend.c` / `cellVideoOut.c` compile (clang-cl `/Zs`). Not run in this repo's port.
