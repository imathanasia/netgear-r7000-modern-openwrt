# What this router can do now

Written for the Netgear R7000 with this build (OpenWrt 25.12, kernel 6.18, `b43-next`).
Project and packaging by **ImAthanasia**.

The scenarios come first because that is what people actually look for. The full feature table
is below it. Every command below was run on the reference unit.

## Practical scenarios

**"The router is bricked / the web UI is gone"** → CFE TFTP netboot the RAM-only
`…-initramfs.trx`; it comes up on the LAN, then `sysupgrade -n` the `.chk`. No serial console,
no vendor firmware needed. Full procedure in [install.md](install.md) (method B) and
[recovery.md](recovery.md). Back up `mtd0` (CFE) and `mtd1` (nvram) first.

**"Ads on every phone, TV and laptop"** → point the LAN DNS at the router and run AdGuard Home
on `:53`. The soft-float build and config are in [services.md](services.md); the image ships the
gates (`adguard-hl`) disabled so it only starts once you install it.

**"I want my home LAN from a hotel"** → install Tailscale and
`tailscale up --advertise-routes=192.168.1.0/24`, then reach `192.168.1.x` directly. No port
forwarding, no DDNS, no exposed services. Kernel WireGuard is available too
(`wireguard-tools` + LuCI protocol support).

**"Plug a USB disk in and share it"** → ksmbd serves SMB3 through LuCI (*Network → Network
Shares*), aria2 downloads onto the disk, smartmontools reports disk health. ext4, exfat and
ntfs3 modules are included.

**"Calls stutter when someone downloads"** → SQM with cake on the uplink
(*Network → SQM*); set your measured line rate and latency stays flat under load.

**"Who is eating the uplink?"** → nlbwmon's per-device table, then vnstat2 for the daily and
monthly totals.

**"It sits behind a PPPoE ONT / needs IPv6 / a DDNS name"** → the image carries PPPoE, IPv6
protocol support and ddns-scripts; configure them in LuCI as on any OpenWrt box.

**"A guest network without giving guests the LAN"** → add a second SSID in
*Network → Wireless* and put it in its own zone. Radios stay off until `wifi-radio on`, so the
guest AP only exists when you ask for it.

**"Just a quiet wired router"** → it boots as an AP/router on `192.168.1.28` with the DHCP
server off and both radios off. Nothing transmits; add ports, shares or VPNs as needed.

## Before and after

| # | Job | Stock R7000 firmware (2013, Linux 2.6) | This image |
|---|---|---|---|
| 1 | **Wi-Fi driver** | closed Broadcom `wl`, no updates | `b43-next` AC-PHY, both BCM4360 radios, VHT80 + DFS |
| 2 | **Usable flash space** | vendor layout leaves the big ML region unused | 122 MB UBI → 96.8 MB `rootfs_data` → 87.6 MB `/overlay` |
| 3 | **Ad blocking for every device** | none | AdGuard Home on `:53` with per-client stats, DoH upstream |
| 4 | **Remote access** | port forwards only | Tailscale subnet router or kernel WireGuard |
| 5 | **USB disk sharing** | vendor ReadySHARE | ksmbd SMB3 + aria2 + SMART, ext4/exfat/ntfs3 |
| 6 | **Latency control** | vendor QoS presets | SQM with cake |
| 7 | **Per-device traffic accounting** | none | nlbwmon + vnstat2 |
| 8 | **Security fixes** | none since Netgear stopped | OpenWrt 25.12 / kernel 6.18.60 |
| 9 | **Recovery** | vendor updater, if it still works | CFE TFTP netboot from a RAM image |
| 10 | **Default behaviour on boot** | radios on, default SSID, default password | radios off, password set, LAN `192.168.1.28`, DHCP off |

## What is in the image

`ksmbd-server`, `aria2`, `nlbwmon`, `vnstat2`, `sqm-scripts`, `banip`, `watchcat`,
`wifischedule`, `ddns-scripts`, `wireguard-tools` + `kmod-wireguard`, `smartmontools`,
`usb-modeswitch` + RNDIS tethering modules, `qrencode` (the router page draws a Wi-Fi QR when a
radio is on), `iperf3`, `socat`, `htop`, `lm-sensors`, `ttyd`, and the ext4/exfat/ntfs3
filesystem modules. AdGuard Home and Tailscale are the two deliberate
soft-float installs — see [services.md](services.md).

## What is deliberately not in it

collectd, mwan3, travelmate, minidlna, usbip, ser2net, p910nd and `luci-app-wireguard` are not
part of this image. They can be added with `opkg` if your use case needs them; the soft-float
constraint applies to prebuilt binaries from outside the OpenWrt feed.

## Measuring it yourself

The Cortex-A9 has no VFP and no hardware NAT, so expect CPU-bound numbers, not datasheet ones:

```sh
iperf3 -c 192.168.1.28 -P 4      # from a wired client
```

The reference unit was validated on an isolated LAN (no upstream internet): Wi-Fi bring-up,
LuCI, storage, DNS and the AdGuard UI/listener were verified; AdGuard's upstream resolution and
filter downloads need a real internet connection to test.
