# HP Z440 Linux ReBAR: Downloads and preparation checklist

Companion: [verified dual A770 16GB walkthrough](docs/hp-z440-linux-rebar-flashrom-dual-arc-a770-2026-10.md).

## Original sources and downloaded files

| Item | Source | Role |
|---|---|---|
| Original Windows-focused Z440/Z640/Z840 mod guide | https://github.com/bibikalka1/HP_Z440_Z640_Z840_BIOS_mod | Original instructions, warnings, hardware jumper reference |
| `patch_bios_v03.ps1` | Same upstream repository | Patch reference used for Linux/Python comparison |
| `reference261_v1.zip` / `FLSHM6002.61` | Same upstream repository | Reference BIOS v02.61 firmware for review; never directly flash another machine's full image |
| `patch_bios_v01.ps1` | Same upstream repository | Different patch revision; not interchangeable with v03 |
| Jumper images `Zx40_jumpers.jpg`, `z440_board.jpg` | Same upstream repository | Identify the actual board labels and jumper positions |
| `IMET9_2026.zip`, `FPTW64.exe` | Original Windows-oriented tools | Used for optional WinPE comparison, **not needed for successful Linux path** |
| `sp146166_b0261.exe` | HP / original repository | BIOS reference package, not ready-to-flash modified firmware |
| ReBarUEFI / ReBarState | https://github.com/xCuri0/ReBarUEFI | ReBAR driver documentation and Linux-native state utility |
| flashrom | https://flashrom.org/ | Linux SPI backup and selected-region operations |
| UEFIExtract / UEFITool | https://github.com/LongSoft/UEFITool | Review firmware structure |
| Python 3, git, cmake, build-essential, unzip, pciutils, dmidecode, xxd, sha256sum, cmp | Ubuntu packages | Build, inspect, and verify |
| Original 16MiB SPI backups | Your **own** workstation | Keep private and off the machine |
| Compatible external programmer and SOIC8 clip | Hardware | Potential recovery from non-POST failure |

Exact upstream commit IDs and file hashes used during the original success were not preserved. Verify the current downloads and record their hashes before trying to reproduce; do not assume the upstream files are unchanged.

## Windows to Linux: verified substitutes

| Windows steps | Linux alternative |
|---|---|
| Administrator CMD | Terminal with `sudo` |
| Intel FPT backup | Linux flashrom full-chip read, matching 16MiB copies |
| PowerShell v03 patch script | Python binary range operation with independent firmware inspection |
| Windows S3 suspend/resume | Linux `systemctl suspend`, tested with board jumper configuration |
| Intel FPT firmware flash | Linux flashrom **BIOS-region-only** write, not the same as an unrestricted whole-chip FPT write |
| Windows ReBarState | Build `ReBarUEFI/ReBarState` with CMake on Linux |
| Windows ReBAR check | `lspci -vv` checks current BAR2 size for each GPU |
| Jumper/Bios menu | OS-independent physical firmware steps |

## Before attempting any firmware change

- Confirm **HP Z440 M60 BIOS v02.61**, SPI device identification, UEFI boot, GPU identities, and auxiliary power.
- Identify E1/E49/E14/E15 physical jumpers against your actual board and the upstream photographs. Record original positions. Never move jumpers with power connected.
- Make **two same-session matching full-chip dumps**, confirm each is 16,777,216 bytes and compare hashes; retain separate **offline** backups.
- Compare `patch_bios_v03.ps1` and reference archive with the verified case study. Do not substitute `v01` or another firmware revision's offsets.
- Verify candidate changes and unchanged platform-specific regions manually, including Descriptor, Intel ME and UEFI NVRAM. Use UEFIExtract or UEFITool. A structural pass does not guarantee bootability.
- Ensure a **real recovery method** exists. The original session demonstrated successful flash and boot, but **no tested rollback after bricking**.
- If the SPI chip, protections, suspend behavior or firmware identity differ, stop rather than forcing a write. **Do not automate SPI flashing.**

## After an independently validated BIOS modification

1. BIOS setup: **Advanced → Performance Options → PCIe MMIO Assignment Mode → 64 Bit**; UEFI boot and CSM off.
2. Build and install Linux ReBarState in persistent storage under `$HOME`/`/usr/local/bin`, **not** ephemeral `/tmp`.
3. Interactive ReBarState value **12** = 4GB; reboot and confirm. Value **14** = 16GB; reboot and confirm.
4. Run `lspci -nn | grep -i 'Arc A770'` and `sudo lspci -vv | grep -E 'Arc A770|Physical Resizable BAR|BAR 2: current size'`. The validated machine showed **04:00.0** and **08:00.0**, both `BAR 2: current size: 16GB`.
5. If only one GPU is present, shut down and confirm the missing card's PCIe auxiliary power; this caused the temporary missing card during the actual test.

## Troubleshooting / stop conditions

| Symptom | Next action |
|---|---|
| Unknown BIOS revision, chip, patch or unexpected backup mismatches | Stop, identify and validate |
| SPI write protection or Linux S3 failure | Stop, inspect platform conditions; do not blindly override locks |
| Modified BIOS boots with 256MB BAR | Verify PCIe MMIO 64 Bit and ReBarState before reflashing |
| ReBarState disappears after reboot | Rebuild in persistent home location |
| No POST | Use pre-planned recovery; CMOS reset may not clear persistent EFI variables |
| BAR2 16GB but no measured speedup | BAR mapping does not imply proportional GPU compute speed |

## 한국어 요약

HP Z440의 Windows 기반 BIOS 개조 원본 프로젝트와 Linux 대체 도구, 다운로드 파일 목록, 작업 전 점검 및 중단 조건을 정리했습니다. 확인된 환경은 HP Z440 / M60 v02.61 / Ubuntu 22.04 / Arc A770 16GB 두 장이며, 두 카드에서 각각 16GB ReBAR 적용이 확인됐습니다. 원본 펌웨어 백업과 안전한 복구 수단 없이 SPI 기록을 시도하지 마세요.