# SenseCAP M1: reproducible Balena deployment

This fork builds the GPIO-compatible packet forwarder, gateway configuration
service and supervised multiplexer from pinned Git submodules. The remaining
service images and named data volumes are preserved. Supported deployments are
the existing ARM64 Raspberry Pi fleets; this does not upgrade balenaOS.

## Build and deploy

```sh
git clone --recurse-submodules --branch repair/balenaos8-reliability https://github.com/Stromer199/helium-sensecap.git
cd helium-sensecap
balena login
balena push julianheger/helium-sensecap --draft
balena device pin DEVICE_UUID RELEASE_COMMIT
balena device logs DEVICE_UUID
```

`balena login` offers browser login. Use the release commit returned by the build.
A draft is available for explicit device pinning without changing other devices.
After hardware startup, radio reception and forwarding are verified, finalize it
with `balena release finalize RELEASE_COMMIT` and explicitly pin the fleet using
`balena fleet pin julianheger/helium-sensecap RELEASE_COMMIT`. Devices with their own pins
must be repinned or returned to fleet tracking. Keep the previous release commit
for rollback using the same `device pin` command. Never purge the data volumes:
they include the hotspot identity and persisted configuration.

When updating an existing checkout, use `git submodule update --init --recursive`.
The source revisions are the committed Git submodule IDs; no mutable packet
forwarder image or uncommitted external checkout is used to compile the radios.

## First upgrade from a legacy sysfs GPIO release

Old containers can leave both the concentrator reset and user-button lines
exported through sysfs after they stop. The new packet forwarder or gateway
configuration service then reports `Device or resource busy`. This is a
one-time migration owned by the fleet operator, only for confirmed legacy
exports. First stop and verify that **both old `packet-forwarder` and old
`gateway-config` containers have stopped**. Keep them stopped throughout the
procedure; a running old service can export the lines again.

The supported default controller offsets are:

| Hardware variant | Reset | Button |
| --- | ---: | ---: |
| SenseCAP M1 (`sensecap-fl1`) | 17 | 27 |
| Nebra Indoor Gen 1 (`nebra-indoor1`) | 38 | 26 |
| RAK (`COMP-RAKHM` / `rak-fl1`) | 25 | 7 |

These are controller offsets, not dynamic sysfs GPIO numbers. The example below
selects SenseCAP M1. Honor any existing explicit pin overrides by verifying and
adjusting `pins` before running it. Run from a shell in the **new packet-forwarder
container**. It uses the same controller discovery as the production reset,
checks every selected line before the first unexport, skips unused lines, and
aborts if any selected line has another consumer. It never requests or pulses
the button or reset lines. Do not add unrelated lines.

```sh
cd /opt
/usr/bin/python3 - <<'PYTHON'
import glob
import os
from pathlib import Path
import gpiod
from pktfwd.reset_gpio import find_gpio_chip

pins = [17, 27]  # reset, button; verified controller offsets for this model
chip_path = find_gpio_chip(gpiod, pins, os.environ.get("CONCENTRATOR_GPIO_CHIP"))
with gpiod.Chip(chip_path) as chip:
    controllers = [Path(path) for path in glob.glob("/sys/class/gpio/gpiochip*")
                   if (Path(path) / "label").read_text().strip() == chip.label()
                   and int((Path(path) / "ngpio").read_text()) == chip.num_lines()]
    if len(controllers) != 1:
        raise SystemExit("Ambiguous sysfs controller; no lines changed")
    base = int((controllers[0] / "base").read_text())
    legacy = []
    for pin in pins:
        line = chip.get_line(pin)
        if not line.is_used():
            continue
        if line.consumer() != "sysfs":
            raise SystemExit("Selected line has another consumer; no lines changed")
        if not Path("/sys/class/gpio/gpio%d" % (base + pin)).is_dir():
            raise SystemExit("Missing selected sysfs export; no lines changed")
        legacy.append(pin)
    for pin in legacy:
        line = chip.get_line(pin)
        line.update()
        if line.consumer() != "sysfs":
            raise SystemExit("Ownership changed; stopped without releasing this line")
        with open("/sys/class/gpio/unexport", "w") as unexport:
            unexport.write(str(base + pin))
        print("Released legacy sysfs export", base + pin, "on", chip_path)
    if not legacy:
        print("No selected legacy exports to release")
PYTHON
```

Never release lines owned by another consumer. After this succeeds, restart the
new `gateway-config` and `packet-forwarder` services and verify their GPIO
startup, real RF packets and forwarding. A cold host boot also clears old sysfs
exports. The production GPIO paths use character devices; no automatic sysfs
migration is installed. Repeat this operator procedure only if a legacy
sysfs-based release is rolled back into use, and remove it once those rollback
releases are retired.

## Behavior and verification

- GPIO controller discovery uses labels and offsets, independent of kernel GPIO
  numbering. Reset/probe failures stop startup instead of choosing a wrong radio.
- Existing 1 ms radio receive polling is preserved. Production payload logging
  is gated by the existing compile-time debug option. The local PUSH_ACK wait
  ceiling defaults to 20 ms (`PKTFWD_PUSH_TIMEOUT_MS`, 2–1000 ms); it returns
  immediately on an ACK. A late ACK may no longer appear in ACK statistics.
- Valid packets preceding a damaged SX1302 receive record are retained. Invalid
  downlink lengths and radio-chain indices are rejected before memory access.
- Missing multiplexer keepalive ACKs trigger recovery; changed destination DNS
  restarts the multiplexer child. Radio region and transmit power limits remain.
- Check both packet-forwarder RF counters and the gateway's received-uplink log.
  A local UDP ACK alone is not proof of a cloud-accepted or rewarded packet.

No thermal protection, firmware clock or voltage configuration is altered by
this release. Higher rewards cannot be guaranteed; actual routed traffic and
current network rules determine eligibility.
