# Intel Arc A770 Linux Idle Power: Runtime PM, D3hot, and Sleep-Aware Monitoring

A practical troubleshooting note for **Intel Arc A770 on a headless Linux server** where the GPU appeared idle but still consumed roughly **34–35 W**.

The key finding was not a broken GPU or failed runtime power management. Runtime PM worked correctly. The main problems were:

1. the GPU was held at `power/control=on`, which prevented runtime suspend; and
2. monitoring code that read GPU hwmon, energy, temperature, RC6, or frequency files could wake the GPU again after it had entered a low-power state.

The stable solution was:

> Keep runtime PM on `auto`, allow the GPU to enter `suspended / D3hot`, and make monitoring **sleep-aware**: if the GPU is already suspended, do not touch sensor/register paths that may wake it.

## Tested setup

This was reproduced on:

- Intel Arc A770 16 GB
- Linux using the `xe` driver
- headless Ubuntu server
- older X99 / Xeon platform
- GPU used for AI workloads such as ComfyUI
- custom dashboards and periodic monitoring

The exact numbers and sysfs layout can vary by kernel, driver, and platform.

## Symptom

The GPU looked computationally idle:

- core frequency near `0 MHz`
- no active AI workload
- no obvious render clients

but GPU power still stayed around **34–35 W**.

Runtime PM initially showed:

```text
power/control: on
runtime_status: active
power_state: D0
runtime_usage: 1
```

## Root cause 1: runtime PM was forbidden

Linux runtime PM semantics are straightforward:

- `power/control=on`: do not runtime-suspend the device
- `power/control=auto`: allow runtime suspend when the device is idle

After switching the A770 to `auto`, the card successfully reached:

```text
power/control: auto
runtime_status: suspended
runtime_usage: 0
power_state: D3hot
```

That proved the hardware and driver were capable of runtime suspend.

### Quick test

First identify the GPU PCI address:

```bash
lspci -nn | grep -Ei 'VGA|Display'
```

Then inspect runtime PM state. Example BDF:

```bash
GPU=0000:05:00.0

cat /sys/bus/pci/devices/$GPU/power/control
cat /sys/bus/pci/devices/$GPU/power/runtime_status
cat /sys/bus/pci/devices/$GPU/power/runtime_usage
cat /sys/bus/pci/devices/$GPU/power_state
```

Temporarily allow runtime suspend:

```bash
echo auto | sudo tee /sys/bus/pci/devices/$GPU/power/control
sleep 3

cat /sys/bus/pci/devices/$GPU/power/runtime_status
cat /sys/bus/pci/devices/$GPU/power_state
```

A successful idle transition may look like:

```text
suspended
D3hot
```

## Root cause 2: monitoring can wake a suspended GPU

This was the less obvious part.

Periodic monitoring originally read paths such as:

- GPU hwmon
- energy counters
- temperature sensors
- RC6 residency
- active frequency
- dashboard hardware details

The GPU could enter `D3hot`, but subsequent sensor polling could bring it back to `D0`.

This creates a misleading loop:

1. monitoring asks "is the GPU sleeping?"
2. monitoring reads a register/sensor that requires the GPU to wake
3. the GPU wakes
4. monitoring reports high idle power
5. it looks like runtime PM is broken

## The sleep-aware monitoring rule

Always read the passive runtime PM state **first**:

```bash
STATUS=$(cat /sys/bus/pci/devices/$GPU/power/runtime_status)
```

If it is `suspended`, stop there.

Do **not** proceed to hwmon, energy, temperature, or frequency reads.

Conceptually:

```python
runtime_status = read("/sys/bus/pci/devices/0000:05:00.0/power/runtime_status")

if runtime_status == "suspended":
    metrics = {
        "runtime_status": "suspended",
        "power_state": read("/sys/bus/pci/devices/0000:05:00.0/power_state"),
        "power_w": None,
        "temperature_c": None,
        "frequency_mhz": None,
        "sensor_reads_skipped": True,
    }
else:
    # Only now read active GPU sensors.
    metrics = read_gpu_sensors()
```

When suspended, `None` / unavailable is more correct than waking the GPU just to produce a number.

## Verified wake/sleep round trip

The following behavior was confirmed:

### Idle

```text
control: auto
runtime_status: suspended
runtime_usage: 0
power_state: D3hot
```

### Start ComfyUI

The GPU automatically resumed:

```text
runtime_status: active
power_state: D0
```

### Stop ComfyUI

After the workload ended, it returned to:

```text
runtime_status: suspended
power_state: D3hot
```

No manual wake command was needed.

## Persist runtime PM across boot

See:

- [scripts/enable-intel-gpu-runtime-pm](scripts/enable-intel-gpu-runtime-pm)
- [systemd/a770-runtime-pm.service](systemd/a770-runtime-pm.service)

The helper discovers an Intel display-class DRM device and writes `auto` to its runtime PM control.

Install example:

```bash
sudo install -m 0755 scripts/enable-intel-gpu-runtime-pm \
  /usr/local/libexec/enable-intel-gpu-runtime-pm

sudo install -m 0644 systemd/a770-runtime-pm.service \
  /etc/systemd/system/a770-runtime-pm.service

sudo systemctl daemon-reload
sudo systemctl enable --now a770-runtime-pm.service
```

## Safe status check

See [scripts/gpu-runtime-status-safe](scripts/gpu-runtime-status-safe).

It intentionally reads only runtime PM-related sysfs fields and does not query hwmon or frequency registers.

Example output:

```text
pci_address=0000:05:00.0
driver=xe
control=auto
runtime_status=suspended
runtime_usage=0
power_state=D3hot
autosuspend_delay_ms=1000
d3cold_allowed=1
```

## Dashboard behavior

A dashboard should not display stale or fabricated GPU power while the device is suspended.

A better UI is:

```text
GPU: Sleeping · D3hot
Power: —
Temperature: —
Frequency: —
Sensor reads skipped
```

When the GPU becomes active again, normal live sensor collection can resume.

## Why this can be difficult on a headless server

Several layers interact:

- BIOS PCIe ASPM settings
- PCIe topology
- kernel runtime PM
- Intel `xe` driver behavior
- systemd automation
- dashboards
- hwmon polling
- AI services that legitimately wake the GPU

An older X99 platform adds another variable because modern GPU low-power behavior is being used on a much older PCIe/firmware platform.

The important lesson from this case is that **successful D3hot entry is not enough**. Every periodic monitor must also avoid waking the card.

## What I would check first

If an Arc GPU stays at unexpectedly high idle power on Linux:

1. Stop actual GPU workloads.
2. Check `power/control`.
3. Set it to `auto` for a temporary test.
4. Check whether `runtime_status` reaches `suspended`.
5. Check whether `power_state` reaches `D3hot`.
6. If it does, inspect monitoring and dashboard processes.
7. Make all GPU polling sleep-aware.
8. Verify workload wake to `D0` and post-workload return to `D3hot`.
9. Only then investigate deeper ASPM/firmware issues.

## What not to conclude too quickly

- A 0 MHz core clock does **not** mean the PCI device is in a low-power state.
- A high hwmon power reading does not necessarily prove the GPU cannot suspend if reading that sensor itself wakes the device.
- Do not assume every Arc idle-power problem is ASPM.
- Avoid forcing aggressive PCIe settings before proving whether normal runtime PM works.

## Korean summary / 한국어 요약

헤드리스 Ubuntu 서버에서 Arc A770이 작업을 하지 않는데도 약 34–35W를 소비하는 문제를 추적했습니다.

핵심 원인은 두 가지였습니다.

1. `power/control=on` 상태라 runtime suspend가 금지되어 있었음.
2. `auto`로 바꿔 D3hot에 들어간 뒤에도 전력/온도/클럭/RC6를 읽는 모니터링 코드가 GPU를 다시 깨움.

최종적으로:

```text
control=auto
runtime_status=suspended
runtime_usage=0
power_state=D3hot
```

상태가 유지되도록 했고, ComfyUI 같은 GPU 작업이 시작되면 자동으로 D0로 복귀하고 작업 종료 후 다시 D3hot으로 내려가는 것도 확인했습니다.

가장 중요한 구현 원칙은 **GPU가 suspended 상태라면 hwmon, energy, frequency, temperature 센서를 읽지 않는 것**입니다.

## Notes

This repository documents one confirmed configuration and debugging path. It is not a guarantee that every Arc idle-power issue has the same cause.

If your GPU cannot reach `D3hot` even with `power/control=auto` and no active clients, then PCIe ASPM, firmware, kernel, driver, audio functions, or platform-specific issues may still need investigation.

## License

MIT


## 2026-09-28 extended findings

Further tracing on the same A770 LE / X99 server found several additional causes of unwanted wakeups and clarified an important limitation:

- **DRM connector polling** was waking the GPU periodically. ftrace showed a path through `output_poll_execute -> intel_dp_detect -> xe_pm_runtime_resume`. Temporarily setting `/sys/module/drm_kms_helper/parameters/poll` to `N` reduced active residency substantially on this headless system.
- A custom idle monitor was reading Xe `act_freq` even while the GPU was already runtime-suspended. ftrace showed `act_freq_show -> xe_pm_runtime_get -> rpm_resume`. The monitor was changed to read `runtime_status` first and skip frequency reads while suspended.
- **ComfyUI holds the render node open while running.** Two open file descriptors to `/dev/dri/renderD128` corresponded to `runtime_usage=2`, keeping the GPU in `D0`. Stopping ComfyUI allowed `runtime_usage=0` and `D3hot`.
- After the wake sources above were removed, runtime active residency fell from roughly **11–12 s/min** to about **1–2 s/min** in the observed tests.
- However, the Xe `energy2_input` package counter still indicated roughly **35 W average** even during long intervals where runtime PM reported almost continuous `D3hot`. This suggests that **runtime D3hot alone does not guarantee low board/package idle power** on this platform.
- PCIe inspection showed the X99 root port had ordinary **ASPM L1** but no visible **L1 Substates capability**, while the A770 internal bridge advertised L1.1/L1.2 support but had those substates disabled. The firmware ACPI FADT also told Linux that ASPM is unsupported, so Linux used BIOS configuration and would not allow changing the ASPM policy at runtime.

See [docs/x99-a770-idle-debug-2026-09-28.md](docs/x99-a770-idle-debug-2026-09-28.md) for the detailed trace and measurements.

### Updated practical conclusion

On this specific X99 platform, the software-side runtime-PM behavior can be made clean and sleep-aware, but the remaining ~35 W package idle reading appears tied to platform/firmware PCIe power-management limitations rather than ordinary userspace polling alone.

Do not assume that forcing `pcie_aspm=force` is safe. It can enable ASPM on links that firmware did not expose as safe and may cause instability. Prefer firmware/BIOS support for Native ASPM and L1 Substates, or a newer platform that exposes those capabilities correctly.


## 2026-10-02 hardware bring-up and migration lessons

A broader follow-up note is available here:

- [Old workstation + Intel Arc A770 headless AI server: hardware bring-up lessons](docs/hardware-bringup-migration-lessons-2026-10-02.md)

It covers the hardware-side lessons that accumulated beyond GPU idle power: failed HP Z440 migration, one-second power cut fault isolation, headless boot assumptions, dynamic PCI/DRM/NIC/storage discovery, PCI subsystem vendor identification, `update-pciids`, driver-reported GPU power limits, kernel fallbacks, and avoiding hardware-specific hard-coding in monitoring software.
