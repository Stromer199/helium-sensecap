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

A previous container may have left the concentrator reset line exported through
sysfs. If the new reset reports `Device or resource busy`, first ensure the old
packet-forwarder container has stopped. Follow the guarded one-time migration in
[packet-forwarder/README.md](packet-forwarder/README.md), using controller offset
17 for this model. It verifies the GPIO controller and `sysfs` ownership before
releasing that one export. Restart the packet-forwarder service and verify real
RF packets and local ACKs. Do not unexport unrelated lines. A cold host boot also
clears old sysfs exports. The production reset uses GPIO character devices only.

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
