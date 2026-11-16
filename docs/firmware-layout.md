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

There is **no second flash volume and no `/mnt/flash`**. On a stock unit the
vendor flash still carries `board_data`/`POT`/`ML` magics and `bcm47xxpart`
marks those partitions read-only; the patch below makes the region writable so
the first-boot UBI setup can format it, after which the remaining space is one
`rootfs_data` volume.

The arithmetic matters:

- `rootfs` (18.4 MiB) holds the base system.
- `rootfs_data` (96.8 MiB) is the only writable volume; ubifs reports 87.6 MiB
  of usable space for `/overlay` after its own overhead.
- Installing AdGuard Home (~36 MB) and Tailscale (~28 MB) under `/opt` fits
  with plenty of room for logs and package updates — but it is not unlimited,
  and `sysupgrade` (without `-n`) has to snapshot it.

## Why the ML partition is writable

On a stock R7000 the flash still contains the vendor `board_data`, `POT` and
`ML` magics. `bcm47xxpart` marks those partitions read-only
(`mask_flags = MTD_WRITEABLE`), so the first-boot UBI setup cannot erase or
format the large `ML` region (`ubiformat` fails with `EROFS`) and the writable
area stays limited to the vendor-sized `firmware` partition.

`patches/0001-mtd-bcm47xxpart-make-ML-writable.patch` clears the read-only mask
for `board_data`, `POT` and `ML`; bootloader, factory and T_Meter areas stay
read-only. Once the image has been written, those vendor magics are gone:
`bcm47xxpart` then finds only `boot`, `nvram` and `firmware`, and the `mtdsplit`
TRX parser splits `firmware` into `linux` plus the 122 MiB `ubi` partition — the
state in the table above, verified in `dmesg` on the reference unit
(`3 bcm47xxpart partitions found`, `0x000000400000-0x000007e00000 : "ubi"`).

## sysupgrade behaviour

| Command | `rootfs` | `rootfs_data` |
|---|---|---|
| `sysupgrade image.chk` | replaced | **kept** (settings, `/opt`, packages) |
| `sysupgrade -n image.chk` | replaced | recreated empty |

The `.chk` wrapper is the Netgear format (`U12H270T00_NETGEAR`); the stock
updater and OpenWrt's platform check both accept it.

## Stock firmware

The stock R7000 firmware keeps its own partition table and its own UBI volumes.
Use Netgear's recovery procedure (CFE TFTP or the vendor updater) with a vendor
`.chk` to go back; do not push a vendor `.trx` through OpenWrt's `sysupgrade`,
which validates the OpenWrt/UBI layout. CFE and nvram are untouched by this
image either way.
