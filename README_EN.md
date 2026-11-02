# Goodix GDIX51C0 Linux fingerprint driver · Huawei MateBook D15 2022

[Detailed investigation and fix (Chinese)](https://xingwangzhe.fun/posts/matebook-d15-goodix-gdix51c0-fingerprint/)

[中文](README.md)

## Hardware manufacturer and tested hosts

**Shenzhen Goodix Technology Co., Ltd. (Goodix) makes the fingerprint sensor; Huawei makes the laptop.** This fork targets the Goodix `GDIX51C0`, enumerated through ACPI and connected over SPI with GPIO interrupt and reset lines. The tested sensor reports chip ID `0x2504`, ChicagoHS, and an 80 × 64 pixel array. Huawei's Windows `gfspi.inf` also matches `ACPI\GDIX51C0`; its DLLs, firmware, and installer are **not** included here.

| Host | Evidence |
| --- | --- |
| Huawei MateBook D15 2022, BoF-XX / M1010 | Right index finger enrollment completed and `fprintd-verify` returned `verify-match`; GDM login after reboot has not been checked |
| Huawei MateBook 16s | Hardware tested by the [upstream project](https://github.com/berkekbgz/libfprint-goodix-spi); not retested on this fork |

## Fix in this fork

The upstream driver keeps the SPI bus quiet for about 80 ms while the sensor captures an image. On BoF-XX, IRQ rises during that interval. The listener consumes the edge while reads are suppressed and then waits for another edge, so TLS image response `0x17` times out. This fork wakes the listener and rechecks the IRQ level when suppression ends. It also starts the quiet period in the thread that receives the capture ACK. On 2026-11-02 the branch also merged upstream `505dc7e`: the udev rule now matches `acpi:GDIX51C0:*` when the firmware advertises a `_CID`, fixing the unbound spidev after reboot, and damaged packet headers are rejected and drained instead of being trusted. The capture quiet flag is armed inside the SPI bus lock so a late ACK cannot enter the quiet period early. See [D15-README_EN.md](D15-README_EN.md) for the local evidence, changed files, and rollback path.

## Install and use

This is a fork of [libfprint-goodix-spi](https://github.com/berkekbgz/libfprint-goodix-spi). The installer fetches pinned libfprint source, builds a shadow library, and configures `fprintd`. The default build also installs a patched fprintd for persistent matcher learning; `--without-fprintd` keeps the distribution daemon.

```bash
./install.sh --without-fprintd
fprintd-enroll -f right-index-finger
fprintd-verify
```

First use requires a TLS PSK. The default installation may generate and provision a new key; Windows and Linux may replace each other's key on a dual-boot machine. Read the upstream [first-run notes](https://github.com/berkekbgz/libfprint-goodix-spi#first-run) first. Images and templates are processed and stored on the host. Fingerprint unlock is a convenience feature and should not be the only authentication factor.

## License and provenance

**This fork is distributed under GNU GPL v3.0 (`GPL-3.0-only`).** The upstream project used `LGPL-2.1-or-later`. Section 3 of LGPL 2.1 permits converting a given copy to the ordinary GPL and selecting version 3. This copy updates source license notices, retains the original copyright holders, and records the conversion and source revisions in [PROVENANCE.md](PROVENANCE.md). The fprintd integration patch originates from `GPL-2.0-or-later` source and is distributed as GPLv3 in this copy. See [LICENSE](LICENSE) for the full text.

No Goodix or Huawei Windows driver binary, firmware, fingerprint capture, template, or device key is included. Publishing this repository does not publish the local `/var/lib/fprint` state.
