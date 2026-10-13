# Windows troubleshooting: out of memory, AMD RDNA2 and online play

Notes from getting version 1.6 to run on one Windows 11 PC: AMD Radeon RX 6900 XT (RDNA2,
16 GB), 32 hardware threads, Xbox 360 controller, 4K display, game 1.09. Each entry gives the
line in `user\last_run.log`, what it means, and the setting that got past it. The settings go
in the launcher (**Advanced**, **Graphics**) or in **Advanced → Extra variables**.

*Out of memory* below comes from a second machine: Windows 11, i5-12400F, 16 GB RAM, RX 6600
(RDNA2, 8 GB), 3440x1440 display, game 1.09, version 1.7. The RDNA2 settings in *Crashes*
were needed there too.

## Crashes

| `last_run.log` | When | Cause | What helped |
|---|---|---|---|
| `Unimplemented PM4 type 0, base reg: 16384, size: 15810` then `STOP: GPU library assertion failed` | Right after a save loads | The GPU thread decoded a command buffer the game had already reused: the size is garbage | `BB_COPY_GPU_BUFFERS=1` (copy command buffers at submit) and **Two-stage GPU pipeline: Off** |
| `vk_scheduler.cpp: SubmitExecution: Device lost during submit` | When the world appears | GPU hang/reset (TDR) | `BB_PREP_WORKERS=0` (no draw-preparation workers) and **Object motion vectors** off |
| `Fault: read access to 0000005300000000 at guest offset 0x263b8e7` | Title screen, Two-stage GPU pipeline on | The game read a pointer before the GPU-side write it waits for had landed | **Two-stage GPU pipeline: Off**. Keeping it on with only its fence and memory-write stages off (`BB_TOGGLE_FILE` with bits 38–42) was not tested to the end |
| `vk_graphics_pipeline.cpp: Failed to create graphics pipeline: ErrorUnknown` | At start, after the crashes above | Pipeline cache written by crashed runs (`WarmUp: N stale pipelines were found` the run before) | **Advanced → Clear shader cache** |

The combination that runs stably on this PC:

```
Two-stage GPU pipeline: Off
Extra variables: BB_COPY_GPU_BUFFERS=1 BB_PREP_WORKERS=0
Object motion vectors: off
```

## Out of memory: the Windows commit limit

Four unrelated-looking crash signatures turn out to be one thing: the process could not get
memory. Each allocator reports its own error, so the crash looks like a different bug every
time and sends you chasing the renderer.

| `last_run.log` | Allocator that was refused |
|---|---|
| `Runtime: cannot create 6080 MiB of guest memory (error 1455)`, then `[Dantelion2 Panic] _AllocateDirectMemory() failed for creating runtime heap` | Win32 commit (error 1455 is `ERROR_COMMITMENT_LIMIT`) |
| `vk_graphics_pipeline.cpp: Failed to create graphics pipeline: ErrorOutOfHostMemory` | Vulkan host memory |
| `vk_platform.h: Check: Assertion Failed!` with `vk::Result=ErrorOutOfDeviceMemory` | Vulkan device memory |
| `liverpool.h: unhandled_exception: Unreachable code!` with `Unhandled exception: std::bad_alloc` | C++ heap |

The giveaway is that the signature *moves* between runs while nothing else changes: whichever
allocator happens to ask next is the one that fails. Freeing a little memory does not fix it,
it just moves the crash later.

Measured on Windows 11, 16 GB RAM, RX 6600 (RDNA2, 8 GB), game 1.09, 1080p output, v1.7:

```
desktop idle                     commit available  8.8  GB
game sitting at the main menu    commit available  2.5  GB
pressing Continue to load a save commit available  0.06 GB  -> std::bad_alloc
```

Loading the save took the process from 1,964 MB to 4,874 MB in about six seconds. The port
needs roughly **9 GB of commit** to be in the world: the guest memory reserved at start, plus
the world data on top.

### Why a PC that runs everything else still fails here

Windows charges the **commit limit** when memory is *reserved*, not when it is touched, and
that limit is physical RAM plus the page file — nothing else. VRAM does not count. Ordinary
games allocate incrementally and stay under physical RAM; this port commits the console's
memory map in one block before the first frame, so it is often the first thing on a machine
ever to need the page file at all. On the PC above, the page file reported
`PeakUsage = 0 MB`: nothing previously run had ever touched it.

Ordinary games also *degrade* when an allocation fails (smaller textures, freed caches). This
port asserts and exits, so the same pressure that costs another game some quality ends the
session here.

### Check it

```powershell
# commit limit and what is left
$m = Get-CimInstance Win32_OperatingSystem
'{0:N1} GB limit, {1:N1} GB available' -f `
  ($m.TotalVirtualMemorySize*1KB/1GB), ($m.FreeVirtualMemory*1KB/1GB)

# page file: two equal numbers means a fixed size that can never grow
(Get-ItemProperty `
  'HKLM:\SYSTEM\CurrentControlSet\Control\Session Manager\Memory Management').PagingFiles
```

Run the first command again **with the game at the main menu**: that is where a marginal
machine shows up, long before it crashes.

### Fix

Set the page file back to system-managed and reboot: `sysdm.cpl` → Advanced → Performance
**Settings** → Advanced → Virtual memory **Change** → **Automatically manage paging file
size**. The kernel only reads this at boot, so the reboot is required.

On the PC above the page file was pinned to a fixed 4 GB (`C:\pagefile.sys 4096 4096`, minimum
equal to maximum). Making it system-managed moved the commit limit from 19.8 GB to 27.3 GB,
and it grows further on demand. Same build, same settings, same save, same keypress:

```
                     before reboot     after reboot
at "Continue"        commit 0.06 GB    commit 9.36 GB
result               std::bad_alloc    playable, 68-76 FPS at 1080p
```

A small fixed page file is a common "optimisation" tweak and is harmless until something
front-loads a large commit. Note also that selecting a 1440p or 4K output makes `run.py`
reserve direct memory automatically (`BB_DMEM_MB=9152`, 8.9 GB), which a 4 GB page file cannot
satisfy at all — so the higher output resolutions fail on such a machine even when 1080p
survives.

## Frame rate

- **FSR 4 at 4K on RDNA2: about 6 FPS.** The log shows
  `FSR 4 v07 INT8 quality model, 2160 tier, output 3840x2160`. The INT8 model is tuned for
  RDNA3; on an RX 6900 XT use **FSR 3.1** (`upscaler=fsr3` in `bbport.ini`).
- The stability settings above cost some speed: the two-stage pipeline is the largest gain
  (20–30 %) when it works.

## Online play

- `HandleLoginReply: Login rejected error code 8`: shadNet error 8 is `LoginInvalidPassword`
  (7 would be an unknown account name). Re-enter the password under **Online co-op**.
- `cpp-httplib failed for GET https://thehuntersdream.com:20443/bb-eu/ss.info: error=10
  (SSL server verification failed)`, shown in the game as a failure to get server
  information. The server's certificate is valid (Let's Encrypt `YE2` → `Root YE` →
  `ISRG Root X2` → `ISRG Root X1`). cpp-httplib checks it twice on Windows: with OpenSSL
  against the Windows `ROOT` and `CA` stores, then with `CertGetCertificateChain` on the leaf
  alone. On this PC the user's intermediate store (`certmgr.msc` → Intermediate Certification
  Authorities) held an expired `ISRG Root X2` cross-certificate (valid until 2025-09-16,
  thumbprint `151682F5218C0A511C28F4060A73B9CA78CE9A53`), and building the chain from the
  leaf alone first failed with `NotTimeValid`. Outside the game both checks pass once Windows
  has cached the current intermediates, yet the game kept failing; the cause inside the game
  is not found yet. Removing the expired certificate is the next thing to try. More detail
  comes with `BB_GPU_LOG=info` in Extra variables.

## Launcher settings get overwritten

The launcher writes `%APPDATA%\bbport-launcher\settings.json` and `bbport.ini` from its own
state when it starts the game or closes. Edits made to those files while it is open are lost.
Close the launcher before editing them by hand, or start the game with `Play Bloodborne.exe`.

## Button layout

**Cheats → Japanese/Asian buttons** (`tweak_circle_confirm`) gives the layout of the Japanese
and Asian releases: Circle confirms and interacts, Cross cancels and dodges, and the button
icons follow. Two things that do not work for this:

- Swapping Cross and Circle in the pad input: the buttons move, but the game still draws the
  Cross icon for interacting.
- The PS4 system setting for the confirm button (`sceSystemServiceParamGetInt` 1000): the game
  asks for it but takes the layout from its region instead.

## Korean text

The game language list has no Korean. With the language left on English, the
[Enhanced + Vanilla Plus Korean translation](https://github.com/kairess/bb-enhanced-vanillaplus-korean)
mod (a Korean font and Korean `engus`/`enggb` message files) shows the game in Korean; put it
in `mods\` below any mods it translates. The *Unlock Game Region* patch is not needed for it.
