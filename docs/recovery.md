# Recovery

The R7000 is hard to brick: CFE stays in `mtd0` and sysupgrade never touches it.
Two independent ways back:

1. **TFTP netboot** — works even with an empty/corrupt `firmware` partition.
2. **Serial console** — works even if the network is misconfigured.

## Backups to take once

```sh
dd if=/dev/mtd0 of=/tmp/mtd0-cfe.bin     # 512 KiB bootloader
dd if=/dev/mtd1 of=/tmp/mtd1-nvram.bin   # 1.5 MiB nvram
```

Store them off the device. A corrupt CFE needs an external SPI-NAND programmer;
with a good dump it can be restored from the initramfs instead.

## TFTP netboot (no serial needed)

The CFE here is a TFTP client with `boot_wait=on`; it requests the file
`vmlinuz` from `192.168.1.2`.

1. Host PC: `192.168.1.2/24`, wired to a LAN port.
2. TFTP root contains the initramfs as `vmlinuz`:

   ```sh
   sudo cp openwrt-bcm53xx-generic-netgear_r7000-initramfs.trx /srv/tftp/vmlinuz
   sudo systemctl start tftpd-hpa
   ```

3. Power-cycle the router. CFE downloads and runs the initramfs in RAM.
4. SSH in (default LAN `192.168.1.28`; generic initramfs may answer on
   `192.168.1.1`), then flash from RAM:

   ```sh
   scp openwrt-bcm53xx-generic-netgear_r7000-squashfs.chk root@192.168.1.28:/tmp/
   ssh root@192.168.1.28
   sysupgrade -n /tmp/openwrt-bcm53xx-generic-netgear_r7000-squashfs.chk
   ```

5. Remove `vmlinuz` from the TFTP root (or stop the server) — otherwise the
   next boot netboots again.

Nothing is written to NAND while netbooting, so a failed attempt costs you
nothing.

## Serial console

3.3 V TTL UART pads on the board, **115200 8N1**, no flow control.

- CFE prompt: interrupt the boot countdown.
- `nvram get boot_wait` should be `on`.
- From the CFE prompt you can re-run TFTP (`tftpboot`, `boot`), inspect
  `nvram show`, or clear a bad `bootcmd` — but do not erase `nvram` casually.

## Boot-loop triage

| Symptom | Likely cause | Fix |
|---|---|---|
| CFE keeps TFTP-booting | `vmlinuz` still in the TFTP root | remove it, power-cycle |
| Kernel boots, no LAN | overlay defaults not applied / wrong subnet | serial, or netboot and check `/etc/config/network` |
| Panic on `mtd` write | flashed a vendor `.trx` bigger than 62 MB | netboot initramfs, write the `.chk` with sysupgrade |
| Radios missing | b43 firmware not loaded | `dmesg | grep b43`, `opkg install kmod-b43-next` |

## Getting back to stock

Use Netgear's own recovery procedure: boot the stock `.chk` from CFE over TFTP
(the same chain as above, with the vendor image as the file CFE boots), or use
the vendor recovery utility. OpenWrt's `sysupgrade` validates images and is not
the supported path back to vendor firmware. CFE and nvram are untouched either
way.
