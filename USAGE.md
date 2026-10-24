# GPU Unlock — usage guide

This is the usage walkthrough for [gpu-unlock.com](https://gpu-unlock.com), the documentation site for the two NVIDIA SM speed-select override modules:

- **Ampere (RTX 30 / GA102–GA106):** [Zagitta/gpu-unlock](https://github.com/Zagitta/gpu-unlock) — the module this repository mirrors.
- **Ada (RTX 40 / AD102, AD103 experimental):** [JustAResearcher/gpu-unlock-ada](https://github.com/JustAResearcher/gpu-unlock-ada)

Every number quoted below is reproduced on the site's [benchmarks page](https://gpu-unlock.com/en/benchmarks) with links back to the upstream evidence files. Nothing on the site or in this guide is extrapolated.

## What it does

The kernel module enables the SM speed-select override before the stock NVIDIA driver binds the card. On Ada (RTX 4090) the signed firmware runs and the payload only touches the two speed-select registers — no driver patching, no VBIOS flash. On Ampere the same technique (V67/PLM) doubles the instruction issue rate of FP/BF16 FMA into FP32 on GA102, GA103, GA104 and GA106.

Linux only. The module must load before the stock NVIDIA driver, which is why the instructions below put it in the initramfs rather than just installing it.

## Requirements

- Linux with a recent kernel (check the module's `Makefile` for the supported range)
- The stock NVIDIA proprietary driver (the module does not replace it)
- Root access for installation
- **Not supported:** Windows (no equivalent pre-driver hook), macOS, and laptops with hybrid graphics where the dGPU is never enumerated at boot

## Install (Ampere, this repository)

```sh
git clone https://github.com/Zagitta/gpu-unlock
cd gpu-unlock
make install
```

Then append `gpu_unlock.targets=all` to your kernel command line:

- **GRUB (Debian/Ubuntu/Mint):** add it inside `GRUB_CMDLINE_LINUX_DEFAULT` in `/etc/default/grub`, then run `sudo update-grub`
- **GRUB (Arch/CachyOS):** same edit, then `sudo grub-mkconfig -o /boot/grub/grub.cfg`
- **Fedora/RHEL:** add to `GRUB_CMDLINE_LINUX` and rebuild the grub config, or use `grubby --update-kernel=ALL --args="gpu_unlock.targets=all"`

Reboot. Verify with:

```sh
lsmod | grep gpu_unlock
dmesg | grep gpu_unlock
```

## Install (Ada, RTX 4090 / RTX 4070 Ti SUPER)

Use the Ada fork:

```sh
git clone https://github.com/JustAResearcher/gpu-unlock-ada
cd gpu-unlock-ada
# follow its README — the build is the same, the initramfs step is mandatory
```

## Initramfs ordering (both modules)

The override must be written before `nvidia` binds the card. On most distributions `make install` handles this; if your distribution is not covered, add the module to your initramfs hooks manually:

- **mkinitcpio (Arch):** add `gpu_unlock` to `MODULES=` in `/etc/mkinitcpio.conf`, before `nvidia`, then `sudo mkinitcpio -P`
- **initramfs-tools (Debian/Ubuntu):** `echo "gpu_unlock" | sudo tee /etc/initramfs-tools/modules` (above the nvidia line), then `sudo update-initramfs -u`
- **dracut (Fedora):** `echo 'add_drivers+=" gpu_unlock "' | sudo tee /etc/dracut.conf.d/gpu-unlock.conf`, then `sudo dracut -f`

## Verify it took effect

1. `dmesg | grep gpu_unlock` — you want a line per target card reporting the unlock applied
2. Run a GEMM benchmark with FP16/BF16 inputs and FP32 accumulation
3. Compare against your stock numbers on the same card — the [benchmarks page](https://gpu-unlock.com/en/benchmarks) shows what moved (tensor f16/f32) and what did not (FP32, INT8, FP8)

## Rollback

- Remove `gpu_unlock.targets=all` from the kernel command line and reboot, or
- `make uninstall` (or uninstall the distribution package), rebuild the initramfs, reboot

The card itself is never modified persistently: the override lives in registers that reset on power cycle.

## Failure modes

| Symptom | Cause | Fix |
| --- | --- | --- |
| Module loads, no dmesg line for your card | PCI ID not in the accepted list | Check the [cards page](https://gpu-unlock.com/en/gpus); unmatched IDs are ignored on purpose |
| `nvidia` fails to bind after install | Module ordering wrong — `nvidia` bound first | Fix initramfs ordering (above); the site's [install page](https://gpu-unlock.com/en/install) has per-distro steps |
| Numbers unchanged after reboot | Override applied before the driver, but the driver re-ran devinit | Report it upstream with your dmesg; known to happen on some VBIOS versions |
| Suspend/resume loses the unlock | Registers reset on resume | Upstream added re-unlock after resume for Ampere; Ada fork tracks the same fix |

## Measured results (from the upstream evidence folders)

- **RTX 4090:** tensor f16/f32 and bf16/f32 up to **2.0x**; FP32, INT8, FP8 unchanged (they were never rate-limited)
- **RTX 4070 Ti SUPER:** measurable tensor gains, smaller than the 4090
- **RTX 3090 Ti (Ampere):** FP16 87 → 118 TFLOPS, BF16 87 → 157 TFLOPS, TF32 42 → 82 TFLOPS on dense 8192³ GEMM at 450 W

Full tables with the rows that did not move: [gpu-unlock.com/en/benchmarks](https://gpu-unlock.com/en/benchmarks).

## Site

- Home: <https://gpu-unlock.com>
- Supported cards: <https://gpu-unlock.com/en/gpus>
- Install walkthrough: <https://gpu-unlock.com/en/install>
- FAQ: <https://gpu-unlock.com/en/faq>
- Machine-readable summary: <https://gpu-unlock.com/llms.txt>
