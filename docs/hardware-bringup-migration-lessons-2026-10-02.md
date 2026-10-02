# Old workstation + Intel Arc A770 headless AI server: hardware bring-up lessons — 2026-10-02

This note collects the hardware-side problems and migration lessons from building a low-cost, headless Ubuntu AI server around an older Xeon/X99 platform and Intel Arc A770 GPUs.

The earlier notes in this repository focused on Arc runtime power management. This one is broader: motherboard swaps, PCIe topology, headless boot assumptions, GPU identity, power-limit handling, network-device renaming, and why hardware-specific values should not be hard-coded into monitoring software.

The goal is not to present a universal build recipe. It is a record of what actually broke, what was verified, and which design changes made the system more portable.

## Current working baseline

The working server at the time of writing is still the original platform:

- Ubuntu Server 22.04 LTS
- Xeon E5-2673 v4
- older X99-class motherboard
- Intel Arc A770 16 GB
- Intel `xe` driver
- headless operation
- NVMe system disk plus SATA HDD storage
- ComfyUI and other AI services started only when needed
- custom monitoring / power-control dashboard
- Tailscale used for remote administration

The kernel currently used for the Arc card is a custom `7.2.8-mi50-xe` build, with `6.8.0-40-generic` kept as a known fallback boot option.

A later attempt to move the existing SSD and GPU into an HP Z440 did **not** complete successfully. The Z440 powered for roughly one second and shut back off when the motherboard power connection was present. Because that fault was not fully isolated, this note treats it as an unresolved PSU / motherboard protection-circuit problem rather than claiming a specific failed component.

## Lesson 1: separate "headless" from "no GPU"

A server does not need a monitor attached in order to boot Linux.

For this build, the intended normal state is:

- GPU installed
- no monitor cable
- system boots
- network comes up
- remote administration starts
- AI workloads use the GPU without a local display

This matters because a missing monitor should not be confused with a one-second power cut or other pre-boot electrical fault.

A machine that starts for about one second and loses power is a different class of problem. Diagnose power delivery, shorts, CPU/VRM power, PSU protection, and minimum hardware configuration before chasing display-related causes.

## Lesson 2: when a board swap fails instantly, reduce to the electrical minimum

The failed Z440 migration produced a useful diagnostic rule.

If the PSU and motherboard are connected and the system spins for about one second before shutting off, start from a true minimum configuration:

1. motherboard
2. CPU
3. CPU cooler
4. one DIMM
5. motherboard main power
6. CPU / memory auxiliary power required by that platform

Disconnect:

- discrete GPU
- SATA disks
- NVMe adapters
- USB accessories
- optional PCIe cards
- nonessential front-panel devices

Then add components back one at a time.

If the minimum configuration still shuts off immediately, the fault domain is much smaller: PSU, motherboard, CPU socket/VRM, power connector/contact, or a chassis short.

For proprietary workstation platforms such as the HP Z440, also distinguish carefully between:

- stock PSU + stock motherboard cabling
- third-party adapters
- modular PSU cables from another model

In this case the PSU was the stock unit, so third-party pinout mismatch was not the leading explanation.

## Lesson 3: do not hard-code PCI addresses, DRM card numbers, or render nodes

During development it is tempting to write things like:

```text
/sys/class/drm/card0
/dev/dri/renderD128
/sys/bus/pci/devices/0000:05:00.0
```

That works until:

- the motherboard changes
- the GPU moves to another slot
- another display device appears
- PCIe enumeration changes
- the kernel assigns a different DRM card number

The monitoring code was changed to discover the Intel display-class device dynamically instead of assuming a fixed BDF or `card0`.

A more portable approach is:

1. scan DRM cards
2. follow their PCI device symlinks
3. check PCI vendor/class
4. identify the Intel display device
5. derive hwmon, driver, render-node, and runtime-PM paths from that device

The same idea applies to storage and networking.

## Lesson 4: network interface names can change after a motherboard swap

The original Netplan configuration was tied to a specific MAC address.

That is fragile during a motherboard migration because the onboard NIC changes.

The configuration was changed from a board-specific MAC match to generic Ethernet-name matching so the server would have a better chance of acquiring networking after a platform change.

This solved the immediate portability problem on the current machine, but broad matches such as `en*` or `eth*` are not ideal for every environment. On a multi-NIC server, a better long-term design is to match the intended interface using stable attributes that will still be valid after the planned migration.

The broader lesson is:

> Before moving a headless Linux installation to another motherboard, inspect Netplan, systemd-networkd, NetworkManager, firewall rules, scripts, and dashboards for hard-coded NIC names or MAC addresses.

A server that boots correctly but has no network can look "dead" when it is actually healthy.

## Lesson 5: derive the default NIC and gateway at runtime

Custom scripts originally contained assumptions about the current interface and gateway.

Those were replaced with runtime discovery:

- default-route interface from `ip route`
- default gateway from the routing table
- NIC PCI device from `/sys/class/net/<iface>/device`

That allows diagnostics such as link state, offload checks, and PCI information to keep working after interface renaming.

Do the same for anything tied to board enumeration.

## Lesson 6: discover HDDs instead of assuming `sda` and `sdb`

Disk letters are not stable identifiers.

The monitoring code was changed to scan `/sys/block`, ignore virtual/non-rotational devices, and identify physical rotational disks dynamically.

This avoids silently reading the wrong device after:

- adding another SATA disk
- moving SATA ports
- changing a storage controller
- booting on another motherboard

For persistent mounts, filesystem UUIDs are preferable to `/dev/sdX` names.

## Lesson 7: GPU board identity is different from GPU chip identity

After replacing the Intel Limited Edition card with a Sparkle A770, the PCI function still identified the GPU chip as Intel DG2 / Arc A770.

That does not mean the board vendor is Intel.

Example after updating the local PCI ID database:

```text
Intel Corporation [8086]
DG2 [Arc A770] [56a0]
Sparkle Computer Co., Ltd. [172f]
Device [3937]
```

This shows three different identity layers:

- GPU silicon vendor: Intel
- GPU family/model: DG2 / Arc A770
- board/subsystem vendor: Sparkle

The dashboard initially hard-coded product-name mappings. That was removed.

The more portable implementation now reads the system PCI database and uses the device and subsystem fields instead of maintaining a private table of card models.

If `lspci` only shows numeric subsystem IDs, refresh the PCI database:

```bash
sudo update-pciids
```

Then inspect the device again:

```bash
lspci -Dnnmm -s <GPU_BDF>
```

This is preferable to embedding mappings such as `172f = Sparkle` in application code.

## Lesson 8: do not hard-code the GPU's maximum power limit

The original dashboard assumed an A770 maximum of 225 W.

That is acceptable for a single fixed card but fails as soon as a different Arc board or GPU is installed.

The power UI was changed to read the driver's reported rated maximum from hwmon, using the appropriate power channel for the active Intel driver.

Conceptually:

```text
power*_rated_max -> card's reported rated maximum
power*_max       -> current configured limit
```

The UI can then derive:

- maximum allowed input
- displayed `/ N W` value
- percentage-of-maximum calculation
- backend validation range

from the detected hardware instead of a constant.

A cached rated maximum is useful while the GPU is runtime-suspended because waking the card only to redraw the dashboard would defeat low-power monitoring.

## Lesson 9: keep power limits persistent, but validate them against the current hardware

A user-selected GPU power limit should survive a reboot, but a saved limit from one GPU should not blindly be applied to a different GPU.

The practical pattern is:

1. detect the current GPU
2. read its rated maximum
3. load the saved requested limit
4. validate the saved value against the detected device
5. apply only if valid
6. otherwise fall back safely

The same principle applies to CPU power limits after a motherboard/CPU change.

Persistence is useful; stale hardware assumptions are not.

## Lesson 10: monitoring software can be a hardware problem

A large part of this build's "hardware debugging" turned out to be software touching hardware at the wrong time.

The A770 could reach:

```text
runtime_status=suspended
runtime_usage=0
power_state=D3hot
```

but sensor polling could wake it again.

The monitoring stack therefore follows this rule:

> Read runtime-PM state first. If the GPU is suspended, do not touch hwmon/frequency/register paths that may resume it.

This is covered in more detail in:

- [README.md](../README.md)
- [x99-a770-idle-debug-2026-09-28.md](x99-a770-idle-debug-2026-09-28.md)

The general hardware lesson is that observability code is not always passive.

## Lesson 11: a 0 MHz GPU clock is not the same thing as a sleeping PCIe device

During debugging the GPU could report essentially no useful core activity while still sitting in `D0` and consuming much more power than expected.

Always distinguish:

- engine/core frequency
- active clients / render-node users
- runtime PM status
- PCI power state
- actual board/package power

A low frequency alone is not evidence of a low-power PCI state.

## Lesson 12: old workstation platforms can expose only part of a modern GPU's power-management stack

The X99 system successfully supports the A770 as an AI accelerator, but PCIe power-management capabilities are not equivalent to those of a modern platform.

Earlier tracing found:

- ordinary ASPM L1 on the X99 root side
- no visible L1 Substates capability on that root port
- L1.1/L1.2 capability present deeper in the A770 topology but disabled
- firmware/ACPI reporting that the OS does not control ASPM policy

The result is an important distinction:

- runtime PM can work
- the GPU can report D3hot
- software wakeups can be eliminated
- yet physical package/board idle power may still be higher than expected

Do not assume that a modern GPU installed in an old workstation will expose every low-power feature available on a current consumer platform.

## Lesson 13: keep a known-good kernel fallback before changing hardware

The server uses a custom Xe-oriented kernel, but a known generic Ubuntu kernel remains installed as a fallback.

Before a motherboard migration:

- verify the fallback kernel image exists
- verify its initramfs exists
- verify GRUB still has a valid entry
- avoid deleting the last known-good kernel just because the current one works

A motherboard swap changes several variables at once: chipset, NIC, USB controllers, PCIe topology, firmware/ACPI behavior, and sometimes storage enumeration.

A fallback boot path is cheap insurance.

## Lesson 14: moving the system SSD does not automatically require reinstalling Linux

The migration plan was to move the existing Ubuntu SSD to the Z440 rather than reinstall.

That is generally a reasonable approach when:

- the root filesystem is portable
- required storage drivers are in the kernel/initramfs
- UEFI boot files are intact
- networking is not tied to the old board
- custom scripts do not assume old PCI paths

The actual Z440 migration did not get far enough to validate a successful Linux boot because the hardware power-cut issue occurred first.

So the lesson is not "the SSD migration was proven." It is:

> Remove board-specific assumptions before the move so that software is not the next failure after the electrical problem is solved.

## Lesson 15: verify current state after every hardware-related code change

A recurring debugging mistake is to edit source code and assume the running daemon is now using it.

For this server, the safe workflow became:

1. make one exact change
2. create a backup
3. run syntax validation
4. restart the relevant service
5. query the live API / hardware state
6. only then call the change active

Examples:

- Python: `python3 -m py_compile`
- JavaScript: `node --check`

Source code on disk and behavior of a long-running process are two different states.

## Practical pre-migration checklist

Before moving a headless Linux AI server to another motherboard:

### Power / physical

- verify PSU and board power connectors
- test minimum hardware configuration first
- remove optional PCIe/storage devices during fault isolation
- check for chassis/standoff shorts
- do not mix modular PSU cables unless pinout compatibility is proven

### Boot

- keep a fallback kernel
- verify UEFI/EFI system partition
- verify GRUB entries
- keep a recent recovery backup

### Network

- remove stale MAC-address binding
- remove fixed interface-name assumptions
- verify firewall rules do not depend on the old NIC name
- make remote access start automatically

### GPU

- discover PCI BDF dynamically
- discover DRM card/render node dynamically
- identify board vendor from subsystem data
- update the PCI ID database when names are missing
- derive power limits from driver-exposed hardware values
- keep monitoring sleep-aware

### Storage

- prefer UUID-based mounts
- discover physical disks dynamically
- do not assume `sda` / `sdb` ordering

### Verification

- check the running kernel
- check GPU driver binding
- check runtime PM
- check network route
- check storage mounts
- check persisted power limits
- restart and verify monitoring services

## Things I would avoid next time

- hard-coding `card0`, `renderD128`, a PCI BDF, a NIC name, or a gateway
- binding a headless server's only network path to the old motherboard MAC before a board swap
- treating `0 MHz` as proof of low GPU power
- polling GPU sensors while the GPU is suspended
- hard-coding a 225 W GPU maximum into a dashboard
- assuming the subsystem vendor will be named correctly with an old `pci.ids`
- assuming a one-second power cut is related to the lack of a monitor
- changing several hardware variables at once without a minimum-config test
- deleting the old kernel before the new hardware has booted successfully

## Current unresolved item

The HP Z440 migration remains unresolved.

Observed symptom:

> With the stock Z440 PSU, connecting motherboard power resulted in approximately one second of fan/power activity followed by shutdown.

At the time of writing, the exact trigger has not been isolated well enough to distinguish confidently between:

- motherboard fault
- PSU protection event
- CPU/VRM-side problem
- connector/contact problem
- chassis short

Until the system is tested in minimum configuration and the failing connection is isolated, it should remain documented as an unresolved electrical/power-protection issue.

## Korean summary / 한국어 요약

저가형 헤드리스 AI 서버를 X99 + Xeon + Intel Arc A770으로 구성하면서 겪은 하드웨어 문제를 정리했습니다.

가장 재사용 가치가 컸던 교훈은 다음과 같습니다.

- 모니터가 없어도 헤드리스 부팅 자체에는 문제가 없으며, 1초 후 전원이 꺼지는 현상은 디스플레이 문제가 아니라 전원/보드 쪽으로 분리해서 봐야 함
- 보드 교체 시 NIC 이름, MAC, PCI 주소, `card0`, `renderD128`, `sda` 같은 값은 바뀔 수 있으므로 가능한 한 자동 탐지해야 함
- Arc A770의 GPU 칩 제조사는 Intel이지만 보드 제조사는 PCI subsystem vendor로 별도 식별해야 함
- `update-pciids` 후 Sparkle 보드가 `Sparkle Computer Co., Ltd. [172f]`로 정상 식별됨
- GPU 최대 전력은 225 W를 코드에 박기보다 드라이버의 `power*_rated_max`를 읽어 UI/검증에 사용해야 함
- GPU가 runtime suspend 상태일 때 센서 폴링을 하면 다시 깨울 수 있으므로 모니터링도 하드웨어 전력상태를 고려해야 함
- X99 같은 구형 플랫폼에서는 runtime D3hot이 정상이어도 PCIe L1 Substates/펌웨어 한계 때문에 실제 유휴 전력이 기대만큼 낮지 않을 수 있음
- 메인보드 이동 전에는 네트워크 하드코딩, PCI 경로 하드코딩, 저장장치 순서 의존성을 미리 제거하는 편이 좋음
- 커스텀 커널을 사용할 때는 새 하드웨어가 완전히 검증될 때까지 범용 fallback 커널을 유지하는 것이 안전함

Z440으로의 실제 이전은 아직 성공하지 못했습니다. 순정 PSU 환경에서 메인보드 전원을 연결하면 약 1초 동작 후 꺼지는 문제가 있으며, 원인을 아직 특정하지 않았기 때문에 이 부분은 추정으로 단정하지 않았습니다.

## Scope

This is a field note from one low-cost home server, not a hardware compatibility guarantee.

The most transferable principle is simple:

> Treat every hardware-derived identifier, limit, path, and power state as something to discover and verify at runtime unless there is a strong reason to make it static.
