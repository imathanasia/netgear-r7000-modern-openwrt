# Install

Three ways in, easiest first. All of them end with the same image:
`release/openwrt-bcm53xx-generic-netgear_r7000-squashfs.chk`.

> The default LAN is **192.168.1.28/24** and the DHCP server is **off**. Your
> computer will not get a lease from this router — set a static address in
> `192.168.1.0/24` (for example `192.168.1.10/24`, gateway `192.168.1.1`) or use
> the initramfs netboot path below. If `192.168.1.28` is already taken on your
> network, pick another host address for your PC and change the router address
> afterwards in LuCI.

## A. From the stock Netgear firmware

1. Connect a PC to a LAN port, open the stock UI (`http://192.168.1.1`).
2. *Advanced → Administration → Firmware Update*.
3. Select the `.chk` file and upload. Do not interrupt power.
4. The router reboots into OpenWrt at `192.168.1.28`; SSH `root` / `changeme`.

The image carries the Netgear board id `U12H270T00_NETGEAR`, so the stock
updater recognises it as a valid R7000 image. If the upload is rejected,
stalls, or the stock build checks signatures, use method B.

## B. Netboot the initramfs, back up, then flash (recommended)

This also gives you the CFE/nvram backups before anything is written.

1. Put the initramfs in a TFTP root on your PC, **named `vmlinuz`** (the CFE
   configuration requests `192.168.1.2:vmlinuz`):

   ```sh
   sudo cp openwrt-bcm53xx-generic-netgear_r7000-initramfs.trx /srv/tftp/vmlinuz
   sudo chmod 644 /srv/tftp/vmlinuz
   ```

2. Give your PC `192.168.1.2/24`, connect it to a LAN port, start the TFTP
   server, and power the router on. `boot_wait=on` makes CFE try TFTP first.

3. The initramfs runs in RAM and brings up the same default LAN
   (`192.168.1.28`). If your host cannot reach it, scan the subnet — a generic
   initramfs sometimes answers on `192.168.1.1` until the overlay defaults run.

4. Back up the bootloader and nvram **now** (they are never rewritten by
   sysupgrade, but a corrupt CFE is unrecoverable without them):

   ```sh
   ssh root@192.168.1.28
   dd if=/dev/mtd0 of=/tmp/cfe.bin
   dd if=/dev/mtd1 of=/tmp/nvram.bin
   # copy both off the device (scp, or a USB stick)
   ```

5. Copy the sysupgrade image over and flash it:

   ```sh
   scp openwrt-bcm53xx-generic-netgear_r7000-squashfs.chk root@192.168.1.28:/tmp/
   ssh root@192.168.1.28
   sysupgrade -n /tmp/openwrt-bcm53xx-generic-netgear_r7000-squashfs.chk
   ```

6. Stop the TFTP server (or remove `vmlinuz`) before the next power-on,
   otherwise CFE will netboot the initramfs again instead of the flash.

## C. From OpenWrt (upgrade or reinstall)

```sh
scp openwrt-bcm53xx-generic-netgear_r7000-squashfs.chk root@<router>:/tmp/
ssh root@<router>
sysupgrade -n /tmp/openwrt-bcm53xx-generic-netgear_r7000-squashfs.chk
```

`-n` discards the overlay; omit it to keep `/overlay` (settings, installed
packages, AdGuard data).

## First boot

```sh
ssh root@192.168.1.28        # changeme
passwd

vi /etc/router-wifi.conf      # SSID / KEY / COUNTRY / CH_2G / CH_5G
wifi-radio on
wifi-radio status
```

Expected state on first boot:

- `/overlay` empty (0% used) and 83 MB free;
- radios registered but APs down (`wifi-radio status` shows the phys);
- dnsmasq listening on `:53`, AdGuard and Tailscale **not** installed.

If the router does not come back, go to [recovery.md](recovery.md).
