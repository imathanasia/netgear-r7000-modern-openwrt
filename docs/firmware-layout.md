# Firmware layout

## Partitions (as seen by Linux)

| MTD | Size | Name | Contents |
|---|---|---|---|
| `mtd0` | 512 KiB | `boot` | CFE bootloader — read-only, never touched |
| `mtd1` | 1.5 MiB | `nvram` | CFE environment (MACs, board id, `boot_wait`) |
| `mtd2` | 126 MiB | `firmware` | kernel + UBI container, written by sysupgrade |
| `mtd3` | ~4 MiB | `linux` | kernel inside `firmware` |
| `mtd4` | 122 MiB | `ubi` | UBI container inside `firmware` |

Offsets: `firmware` starts at `0x200000`; `ubi` starts 4 MiB later at
`0x600000` (absolute).

## UBI inside `ubi`

```
ubi0: 973 LEBs × 124 KiB = 117.8 MiB   (3 bad PEBs)
├── rootfs      152 LEBs  18.4 MiB   squashfs, read-only
└── rootfs_data 800 LEBs  96.8 MiB   ubifs → mounted at /overlay (87.6 MiB usable)
```

There is **no second flash volume and no `/mnt/flash`**. The vendor firmware
carves the end of the NAND into a small `board_data` area that OpenWrt used to
leave read-only and unused; this image makes the whole remaining UBI available
to one `rootfs_data` volume instead.

The arithmetic matters:

- `rootfs` (18.4 MiB) holds the base system.
- `rootfs_data` (96.8 MiB) is the only writable volume; ubifs reports 87.6 MiB
  of usable space for `/overlay` after its own overhead.
- Installing AdGuard Home (~36 MB) and Tailscale (~28 MB) under `/opt` fits
  with plenty of room for logs and package updates — but it is not unlimited,
  and `sysupgrade` (without `-n`) has to snapshot it.

## Why the ML partition is writable

`bcm47xxpart` normally forces the "ML" (manufacturer/data) area read-only, which
prevents UBI from attaching to the full container. The patch in
`patches/0001-mtd-bcm47xxpart-make-ML-writable.patch` clears the read-only mask
for `board_data`, `POT` and `ML`, so the parser can present one contiguous UBI
device. Bootloader, factory and T_Meter areas stay read-only.

After the first boot with the new layout, the parser renames/resizes partitions
on the next mtd write; the table above is the resulting state.

## sysupgrade behaviour

| Command | `rootfs` | `rootfs_data` |
|---|---|---|
| `sysupgrade image.chk` | replaced | **kept** (settings, `/opt`, packages) |
| `sysupgrade -n image.chk` | replaced | recreated empty |

The `.chk` wrapper is the Netgear format (`U12H270T00_NETGEAR`); the stock
updater and OpenWrt's platform check both accept it.

## Stock firmware

The stock R7000 firmware uses the same `firmware` partition but a different
internal split (a vendor TRX plus its own UBI volumes). Flashing a full stock
`.trx` over the 62 MB region overruns the partition and panics — always use a
`.chk` built for this layout, or the stock recovery procedure.
