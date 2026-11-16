# Limitations

Honest list. None of these are surprises; they are the cost of an open driver
and a snapshot build.

## Hardware / CPU

- **No VFP.** The Cortex-A9 in the BCM4709A0 has no floating-point unit. Every
  Go/compiled binary you add must be soft-float (`GOARM=5`), otherwise it dies
  with `Illegal instruction`. Packages from the OpenWrt feed for
  `arm_cortex-a9` match.
- **CPU-bound networking.** No hardware NAT/flow offload in this target; routed
  throughput and USB 3.0 storage are limited by the CPU. Measure with
  `iperf3` on your own clients.
- 256 MB RAM total; AdGuard Home plus Samba plus a download manager can get
  tight (the router page shows live RAM use).

## Wi-Fi (b43-next)

- Experimental out-of-tree driver. It works well on this unit, but it is not
  the vendor `wl` driver: no 160 MHz, no MU-MIMO, no beamforming, no vendor
  proprietary features.
- Only the two seeded profiles are automated: 2.4 GHz HT20 and 5 GHz VHT80.
  Other combinations can be set in LuCI, but are less tested.
- DFS channels need a 60-second CAC before the AP appears.
- WPA2-PSK (`psk2`) is what the seeded config uses. WPA3 is not validated.
- `iw` is absent on purpose; `iwinfo` is the supported tool.
- Radios transmit only after `wifi-radio on`.

## Storage / overlay

- 87.6 MiB usable `/overlay`, shared by settings, AdGuard data, filters, logs
  and any extra packages. AdGuard's default lists plus query logs can grow;
  prune or move them.
- Bad PEBs are normal on this NAND (3 on the reference unit); UBI handles them.

## Software choices

- AdGuard Home and Tailscale are not in the image (licence, size, soft-float
  release cadence). Install them yourself — see
  [services.md](services.md).
- No vendor UI, Netgear Genie, ReadySHARE or vendor cloud features.
- The OpenWrt base is `SNAPSHOT` (`r0-25058cf`). `opkg` feeds move; keep a copy
  of the included `.manifest` if you need to reproduce the exact package set.
- No `luci-app-wireguard` in this build (WireGuard itself is available).

## Recovery

- The serial pinout is not documented here; the TFTP netboot path needs no
  serial and is the primary recovery method.
- Keep your `mtd0`/`mtd1` dumps. A destroyed CFE cannot be restored over the
  network.

## Test rig note

The reference unit was validated with no upstream internet (LAN-only). Wi-Fi
bring-up, LuCI, routing, storage and AdGuard's UI/DNS listener were verified;
AdGuard upstream resolution and filter downloads could not be, because
`192.168.1.1` and public resolvers were unreachable from the test network.
That is a lab limitation, not a firmware defect.
