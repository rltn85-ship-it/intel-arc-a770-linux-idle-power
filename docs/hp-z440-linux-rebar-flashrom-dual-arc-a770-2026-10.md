# HP Z440 ReBAR on Linux: BIOS mod + flashrom + Intel Arc A770 16 GB × 2 (confirmed)

**Tested October 24, 2026 (KST) | Ubuntu 22.04 | HP Z440 M60 v02.61 | Dual Intel Arc A770**

Search terms: **HP Z440 Resizable BAR Linux**, **HP Z440 ReBAR Ubuntu**, **Z440 Above 4G Decoding**, **Z440 PCIe MMIO Assignment Mode 64 Bit**, **HP Z440 flashrom**, **Arc A770 16GB BAR2**, **Z640 Z840 ReBarUEFI**.

> **Confirmed result:** both GPUs enumerated and reported `BAR 2: current size: 16GB` in Linux. This is a *case report*, not a promise that the procedure will work on all firmware revisions or Z640/Z840 systems.
>
> **Danger:** A bad SPI firmware write can permanently prevent boot. A separate SPI programmer, compatible 3.3 V SOIC8 clip and verified recovery image may be required. **Do not flash somebody else's complete firmware dump**: it can overwrite machine-specific NVRAM, MAC/serial/ME data. Do not blindly run the flash commands below.

[한국어 요약](#한국어-요약) · [Tested hardware](#tested-system) · [Linux procedure](#linux-procedure-and-decision-points) · [Result](#verified-results) · [Failure modes](#troubleshooting-and-gotchas)

## What was different about this attempt?

The existing HP Zx40 mod project primarily documents a **Windows + Intel FPT** workflow. Here, the BIOS image was analyzed/patched and the **BIOS region was successfully written using Linux `flashrom`**, after a Linux S3 suspend/resume cycle and the correct physical jumper configuration. The `ReBarState` utility was built and run natively on Linux.

We did **not** port Intel FPT to Linux. A Windows PE/FPT dump was also taken for comparison during exploration, but the successful firmware write was through Linux `flashrom`.

## Tested system

| Item | Observed configuration |
| --- | --- |
| Workstation | **HP Z440**, Intel C610/X99 (Wellsburg) |
| BIOS | HP M60 **v02.61**, release shown as 2023-03-23 |
| CPU | Xeon E5-2673 v4 |
| OS | Ubuntu **22.04 LTS** |
| Running kernel at test | `6.18.56-061856-generic` (non-stock kernel) |
| GPUs | **2 × Intel Arc A770 16 GB**, PCI IDs `8086:56a0` |
| PCI locations at final test | `04:00.0` and `08:00.0` |
| GPU driver observed earlier | Linux `i915` (other tests on this workstation used `xe`; driver is not assumed identical across boots) |
| Boot | UEFI, CSM/Legacy disabled, Secure Boot disabled in this test |
| Kernel option used | `pci=realloc` |
| BIOS setting that mattered | **Advanced → Performance Options → PCIe MMIO Assignment Mode → 64 Bit** |
| Before | BAR2 **256 MB**, although GPU supported up to 16 GB |
| After | BAR2 **16 GB on both A770 cards** |

The host was headless during normal operation. Note that some A770 board variants did not reliably display the firmware setup/POST screen; switching to the other A770 made BIOS output usable in this test. Display issues are separate from BAR allocation.

## Background and references

* **[bibikalka1/HP_Z440_Z640_Z840_BIOS_mod](https://github.com/bibikalka1/HP_Z440_Z640_Z840_BIOS_mod)** — source Zx40 modified-firmware approach, HP jumper photographs, reference 2.61 image and PowerShell patch script. **Credit belongs to the upstream maintainers/contributors.** Review its current instructions and compatibility warnings first.
* **[xCuri0/ReBarUEFI](https://github.com/xCuri0/ReBarUEFI)** — ReBar DXE driver, `ReBarState` Linux/CMake source, Above 4G/CSM guidance.
* **[flashrom](https://flashrom.org/)** — Linux firmware access and Intel flash descriptor region support.

**Compatibility boundary:** The upstream reference and patch were intended for **BIOS 2.61**. Do not apply these exact offsets to a different firmware revision without independent reverse engineering and validation. Do not assume the Z640/Z840 jumpers or address map are identical merely because they are in the upstream project.

## Linux procedure and decision points

### 0. Inspect before modifying anything

```bash
sudo dmidecode -s bios-version
uname -a
lspci -nn | grep -i 'Arc A770'
sudo lspci -vv | grep -A 2 'Physical Resizable BAR'
cat /proc/cmdline
cat /sys/power/mem_sleep
```

Typical original symptom:

```text
Capabilities: [420 v1] Physical Resizable BAR
        BAR 2: current size: 256MB, supported: 256MB 512MB 1GB 2GB 4GB 8GB 16GB
```

The capability alone **does not** mean the larger BAR has been assigned. Earlier kernel logs also complained about insufficient PCI resources and `Can't resize LMEM BAR - platform support is missing`.

### 1. Recovery plan **before** flashing

1. Obtain a working **external SPI programmer/recovery method** or accept the risk of an unrecoverable machine.
2. Verify your exact BIOS version. Make **multiple independent full-chip 16 MiB dumps** and store copies off the Z440. Use `sha256sum` to compare **same-session** dumps.
3. Record the original physical positions of **ME/AMT Flash Override**, green **Password**, and **BBR/CMOS** jumpers with your own photos. Do not confuse them.
4. Ensure stable power, local/remote console recovery, and physical access.
5. Keep your **original full-chip backup private**. Firmware images can contain unique machine data.
6. Validate the firmware layout/diffs before writing; a successful `flashrom` verify is not proof that the firmware is semantically bootable.

**Physical jumper behavior on the tested Z440:** The **E1 ME/AMT Flash Override** jumper was moved to its override position; the **green Password jumper** was temporarily moved from **E49 to E14**; the black **E15/BBR** jumper was left untouched. **Verify labels against your board revision and the upstream photos** — don't rely on colors/positions alone. Return jumpers to their original positions after the procedure, with the power disconnected. This is a machine-specific observation, not universal HP guidance.

### 2. Read the SPI chip from Ubuntu

With the flash override correctly configured, Linux was able to read the installed **Winbond W25Q128.V**, 16 MiB device:

```bash
sudo flashrom -p internal
sudo flashrom -p internal -c W25Q128.V -r z440-original-a.bin
sudo flashrom -p internal -c W25Q128.V -r z440-original-b.bin
sha256sum z440-original-{a,b}.bin
stat -c '%n %s bytes' z440-original-*.bin
```

**The tested original reads matched** SHA-256:
`ca89a55a4c823fea08dc1e004d517053af916c038afee07ca08d49418384cc83`.

These names are illustrative; keep originals immutable. If chip autodetection or the model name differs, **stop and identify the hardware** rather than forcing `-c W25Q128.V`.

A separate Windows PE `FPTW64.exe -d backup.bin` dump was compared during research. That FPT dump and a Linux dump differed in the firmware NVRAM range approximately `0xCA0000–0xD00000` after separate boots; **do not demand bit-identical firmware dumps across sessions with different UEFI variable states**.

### 3. Build a patched BIOS image without destroying unique regions

From the upstream **HP_Z440_Z640_Z840_BIOS_mod** project, review **`reference261_v1.zip`** and **`patch_bios_v03.ps1`**. We reproduced the relevant copy-range logic in Linux/Python, starting from a **local backup** and copying selected reference-image ranges.

The observed `v03` copy ranges were (half-open `[start,end)` offsets in a 16 MiB file):

```python
COPY_RANGES = [
    (0x610010, 0x6A1000),
    (0x6A1000, 0xCA0000),
    (0xD39000, 0xE80000),
    (0xEB0000, 0xFFE000),
    (0xD01000, 0xD31000),
    (0xE80000, 0xEB0000),
    (0xFFE000, 0x1000000),
]
```

**Important:** This is an audit trail, **not a future-proof patch recipe**. Download and read the **current upstream script** before using these offsets. The reference zip contains a full 16 MiB image called `FLSHM6002.61`. Preserve your own descriptor/ME/NVRAM and other system-specific regions; do not flash the reference image directly.

A safe *local construction only* example, after independently validating the ranges and extracting the correct reference:

```python
from pathlib import Path
base = Path("my-own-16MiB-backup.bin").read_bytes()
ref  = Path("FLSHM6002.61").read_bytes()
assert len(base) == len(ref) == 16 * 1024 * 1024
COPY_RANGES = [
    (0x610010,0x6A1000), (0x6A1000,0xCA0000),
    (0xD39000,0xE80000), (0xEB0000,0xFFE000),
    (0xD01000,0xD31000), (0xE80000,0xEB0000),
    (0xFFE000,0x1000000),
]
out = bytearray(base)
for start, end in COPY_RANGES:
    out[start:end] = ref[start:end]
assert out[:0x610010] == base[:0x610010]
assert out[0xCA0000:0xD01000] == base[0xCA0000:0xD01000]
assert out[0xD31000:0xD39000] == base[0xD31000:0xD39000]
Path("candidate-review-only-NOT-FOR-FLASH.bin").write_bytes(out)
```

The actual candidate was checked for 16 MiB length, compared range-by-range with the own backup/reference, and analyzed with **UEFIExtract** before writing. The tested diff contained ~3.86 million changed bytes. The checks did **not** reveal additional structure errors relative to the original dump, but such checks cannot guarantee bootability.

**Do not treat this example as authorization to flash**. Firmware-version matching, region map, boot protections and recovery must all be verified for your own machine.

### 4. Linux S3 resume and SPI write access — what was observed

In the successful session:

* The original internal flashing attempt encountered platform protection (including BIOS lock / write-protected regions).
* With the documented override jumper positions, **Linux S3 suspend/resume** was used before the write, analogous to the Windows-based workflow:
  ```bash
  cat /sys/power/mem_sleep
  sudo systemctl suspend
  ```
  The machine resumed from `deep` (S3).
* After resume, diagnostics showed **BIOS_CNTL=0x09** (write enabled / BIOS lock no longer active) and **PR0–PR4 = 0x0** in that observed session. The Intel descriptor remained locked (`FLOCKDN=1`), so the write was constrained to the firmware's BIOS region.

The **historical successful** command, for reference only, was:

```bash
# DANGEROUS EXAMPLE — DO NOT COPY/PASTE WITHOUT INDEPENDENT VERIFICATION
sudo flashrom -p internal -c W25Q128.V --ifd -i bios --noverify-all \
  -w /path/to/my-validated-16MiB-candidate.bin
```

The original session output included `Erase/write done. Verifying flash... VERIFIED.` and the Z440 subsequently booted. **This proves one tested write path only.** The option `--noverify-all` was part of the historical command, and must not be taken as a recommendation to weaken verification. Use the flashrom documentation appropriate to your build, preserve the untouched descriptor/ME regions, and have recovery ready. A current distro or firmware may enforce different security restrictions; do **not** force flashing past unknown protections.

After power off, physically restore temporarily moved jumpers to their documented original positions. The available test log confirms the successful flash/boot, but **does not independently verify the final jumper-restoration state**.

### 5. Critical BIOS setting: **PCIe MMIO Assignment Mode = 64 Bit**

A modified BIOS **alone was not enough**: BAR2 initially remained **256MB** after flashing.

On the Z440 test machine the key BIOS menu path was:

**Advanced → Performance Options → PCIe MMIO Assignment Mode → `64 Bit`**

Previously it was `Auto`. The firmware did not expose a conveniently named **Above 4G Decoding** switch; setting **64 Bit** enabled high PCIe MMIO allocation in this case. Keep **UEFI boot / CSM disabled**.

Check high-address PCI allocation from Ubuntu:

```bash
grep 'PCI Bus' /proc/iomem
```

The successful test showed a high MMIO area starting at `380000000000` (example, not a required fixed address). Some earlier failures had PCI resource/bridge "no space" messages. Also inspect `sudo dmesg | grep -Ei 'BAR|resource|no space|realloc' | tail -80` if allocation still fails.

### 6. Build and run ReBarState **on Linux**

ReBarUEFI explicitly supports building **ReBarState with CMake on Linux**. Use a persistent directory **in your home**, not `/tmp`: the first attempt lost both source and binary after reboot.

```bash
sudo apt-get install -y git cmake build-essential
git clone --depth 1 https://github.com/xCuri0/ReBarUEFI.git "$HOME/ReBarUEFI"
cmake -S "$HOME/ReBarUEFI/ReBarState" -B "$HOME/rebar-build"
cmake --build "$HOME/rebar-build" -j 4
sudo install -m 0755 "$HOME/rebar-build/ReBarState" /usr/local/bin/ReBarState
sudo ReBarState
```

ReBarState printed:

```text
Current ReBarState 12 / 4096 MB
Enter ReBarState Value
      0: Disabled
Above 0: Maximum BAR size set to 2^x MB
     32: Unlimited BAR size
```

This test used a **staged rollout**:

* Enter **`12`** → `4096 MB`, reboot → **4GB BAR2 verified**.
* Enter **`14`** → `16384 MB`, reboot → **16GB BAR2 verified for BOTH cards**.

Do not type `14` at the Bash prompt: it is an **interactive answer to the ReBarState tool**, not a command.

The firmware must support the allocation; if the next boot fails, revert to the last verified value using a recovery path appropriate to your machine. **Do not assume that clearing CMOS will necessarily remove a persistent UEFI NVRAM variable**. The tested result used `14`, not the unlimited `32` setting.

### 7. Final Linux verification: two Arc A770 16GB GPUs

```bash
lspci -nn | grep -i 'Arc A770'
sudo lspci -vv | grep -E 'Arc A770|Physical Resizable BAR|BAR 2: current size'
```

**Actual final output from this HP Z440 (2026-10-24 KST):**

```text
04:00.0 VGA compatible controller: Intel Corporation DG2 [Arc A770] (rev 08) (prog-if 00 [VGA controller])
        Capabilities: [420 v1] Physical Resizable BAR
                BAR 2: current size: 16GB, supported: 256MB 512MB 1GB 2GB 4GB 8GB 16GB
08:00.0 VGA compatible controller: Intel Corporation DG2 [Arc A770] (rev 08) (prog-if 00 [VGA controller])
        Capabilities: [420 v1] Physical Resizable BAR
                BAR 2: current size: 16GB, supported: 256MB 512MB 1GB 2GB 4GB 8GB 16GB
```

The second GPU was temporarily absent from `lspci` **because its auxiliary PCIe power cable had been intentionally disconnected** during testing. Reconnecting it **with power off** restored enumeration and confirmed 16GB on both GPUs. Do not misdiagnose a disconnected card as a firmware/ReBAR issue.

**Meaning:** BAR aperture per GPU is 16GB, *not* 16GB of additional VRAM or a 64× compute speedup. No controlled before/after AI or gaming benchmark is included here; do not extrapolate performance from BAR size alone.

## Troubleshooting and gotchas

| Symptom | First thing to check |
| --- | --- |
| ReBAR capability exists but current size is 256MB | BIOS mod, `PCIe MMIO Assignment Mode=64 Bit`, CSM off, `ReBarState`, PCI resource logs |
| BIOS mod flashes and boots, but BAR remains 256MB | **Don't flash again immediately**. Check the 64-bit MMIO setting first |
| `ReBarState: command not found` after reboot | Old binary was placed in ephemeral `/tmp`; rebuild/install to `/usr/local/bin` |
| `14: command not found` | Run `sudo ReBarState` first; enter 14 **at its prompt** |
| Only one A770 appears | Confirm **both GPU auxiliary PCIe power connectors**, slot seating and physical installation with AC power disconnected |
| BIOS screen missing or garbled | Alternate A770 board/port, UEFI/GOP/CSM state; don't assume firmware is bricked if OS still boots |
| Flashing denied | Review jumper state, board/descriptor locks and official procedures; **do not disable arbitrary protections** |
| After changing ReBarState, system doesn't POST | Use known-good recovery route; reset/ReBarState rollback may require firmware/NVRAM access |
| Linux 64-bit MMIO not allocated | Check `PCIe MMIO Assignment Mode`, `/proc/iomem`, `dmesg`, boot configuration |
| ReBAR works on one GPU | Verify every physical GPU individually with `lspci -vv -s 04:00.0` and `-s 08:00.0` |
| Network DHCP fails following firmware work | In this test an upstream networking device power cycle resolved it; it was **not** established as a ReBAR problem |

## What this report does **not** prove

* Not a generic guaranteed flash method for any Z440 BIOS, Z640 or Z840.
* Not evidence that PCIe ReBAR improves every ComfyUI/LLM model or that both GPUs combine into one VRAM pool.
* Not a validated safe automated BIOS patcher.
* Not a claim that Windows FPT was ported to Linux.
* Not proof that previously observed GPU idle-power issues disappeared after enabling ReBAR.
* Not an independent verification that the temporary board jumpers were eventually restored.

## 한국어 요약

**HP Z440 / Ubuntu 22.04 / BIOS M60 02.61 / Xeon E5-2673 v4 / Intel Arc A770 16GB × 2**에서 Linux 기반으로 BIOS 개조와 ReBAR 활성화에 성공한 사례입니다.

기존 [HP Zx40 BIOS mod 프로젝트](https://github.com/bibikalka1/HP_Z440_Z640_Z840_BIOS_mod)의 **Windows + FPT 절차를 Linux의 `flashrom` + S3 절전 복귀 + Python 펌웨어 범위 패치 + Linux 빌드 `ReBarState`**로 대체해 검증했습니다. FPT 프로그램 자체를 Linux로 이식한 것은 아닙니다.

**가장 중요했던 단계**는 개조 BIOS 적용 후 **BIOS → Advanced → Performance Options → PCIe MMIO Assignment Mode → `64 Bit`**로 바꾸는 것이었습니다. BIOS 개조만으로는 BAR가 계속 **256MB**였고, 64비트 MMIO 설정 및 `ReBarState` 값 **12(4GB)** 테스트를 거쳐 **14(16GB)**로 올린 뒤 두 GPU의 BAR2가 각각 **16GB**로 표시됐습니다.

필수 주의사항: **BIOS 02.61 대상 수정 범위를 다른 버전에 그대로 적용하지 마세요.** 원본 16MiB SPI 백업 여러 벌과 외부 복구 수단을 확보하고, E1(ME/AMT Flash Override)·E49/E14(비밀번호)·E15(BBR) 점퍼의 실제 보드 표기를 확인하세요. 실패하면 부팅 불가 상태가 될 수 있습니다.

테스트에서 GPU 두 번째 카드가 안 보였던 것은 **보조전원 케이블을 뽑아 둔 것** 때문이었고 재연결 후 두 장 모두 정상 인식됐습니다. 본 사례는 **ReBAR 활성화 확인**까지이며, AI 추론/이미지 생성 성능 향상률은 아직 실측하지 않았습니다.

If this helped you on a Z440/Z640/Z840 Linux build, please share your **BIOS version, distro/kernel, chipset, GPU PCI IDs, `lspci` BAR2 results, and the exact failure point** (never share your raw private full-chip dump).
