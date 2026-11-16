# Discoveries

What this port established, in case it helps the next BCM4709/BCM4360 board.

## The unused 122 MB is usable

Upstream OpenWrt presents the R7000's `ubi` partition but leaves most of it
unattached: the `ML` area is forced read-only by `bcm47xxpart`, so UBI can only
use the vendor-sized container. Clearing the read-only mask for `board_data`,
`POT` and `ML` (while leaving boot/factory/T_Meter alone) lets UBI attach to the
full 122 MiB. The result is a single `rootfs_data` volume of 800 LEBs — no
second partition, no `/mnt/flash` mount, no moving files after boot.

`bcm47xxpart` also rewrites/resizes the partition table on the first mtd write,
so the layout you see before the first sysupgrade is not the final one.

## b43-next AC-PHY on BCM4360 works

The `b43-ac-phy` tree plus `b43-next` at
`ee7ed443dc8dde43bd2b2e9aea7d170ad5e92e1b` brings up both BCM4360 radios
(d11 core rev 42, AC-PHY rev 1, radio 0x2069 rev 4) with:

- HT20 on 2.4 GHz and VHT80 on 5 GHz;
- DFS with `CONFIG_B43_NEXT_DFS=y` (CAC completes; AP starts afterwards);
- the firmware squashed to core rev 42 / AC, which cuts the image size.

No vendor `wl` blob is loaded.

## Firmware auto-enables every init script

`prepare_rootfs` in `include/rootfs.mk` runs `rc.common enable` for **every**
`#!/bin/sh /etc/rc.common` script in the target root, because the image build
calls it without a disable list. Optional services therefore start on first
boot unless they opt out in `/etc/uci-defaults`; this image uses
`95-wifi-off.sh`, `96-tailscale-off.sh` and `97-adguard-off.sh` for that. A
staging-cache purge alone is not a fix — the next image build recreates the
symlinks.

## CFE semantics

- `boot_wait=on` and the CFE configuration
  `Loader:raw Filesys:tftp File:192.168.1.2:vmlinuz` mean the bootloader always
  tries TFTP first. Any TFTP server on `192.168.1.2` wins over flash.
- The requested file name is literally `vmlinuz`; rename the initramfs
  accordingly.
- Netboot writes nothing, so it is a zero-risk way to dump `mtd0`/`mtd1` before
  the first flash.

## No VFP, no hard-float Go

`readelf -A` on a Cortex-A9 build shows no VFP tags here. Go binaries built for
`armv7` crash with `Illegal instruction`; the soft-float (`GOARM=5`) AdGuard
Home and Tailscale builds run correctly. If a prebuilt tool segfaults or SIGILLs
on first run, this is the first thing to check.

## Small runtime details

- Only `iwinfo` is installed; scripts that call `iw` fail. `wifi-radio status`
  falls back to `iwinfo`.
- `dnsmasq` must be moved off `:53` (`dhcp.@dnsmasq[0].port=0`) before AdGuard
  Home starts, otherwise AdGuard exits on bind and procd respawns it silently.
- The generated AP interfaces (`default_radioN`) are disabled; the driver
  registers the phys but nothing is broadcast until a service enables them.
- LuCI 25.12's `config redirect` sections do not redirect here; hidden
  `rewrite`/`alias` nodes do, which is how the legacy service path keeps
  working.
