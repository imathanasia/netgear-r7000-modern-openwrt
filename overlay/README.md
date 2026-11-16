# overlay

This directory is the OpenWrt `files/` overlay used by the image. It is
**board-neutral**: only the netgear branch of the board switches executes on the
R7000, and the other board's aliases are skipped there. It is kept identical to the
tested image so that a rebuild matches the released `.chk` byte layout.

Copy it in with:

```sh
cp -a overlay/. /path/to/openwrt/files/
```

## What is here

| Path | Purpose |
|---|---|
| `etc/uci-defaults/89-uhttpd-luci` | LuCI on uhttpd, sane timeouts |
| `90-rootpass.sh` | default root password `changeme` (change it) |
| `91-branding.sh` | per-board `DISTRIB_DESCRIPTION` |
| `92-theme.sh` | default theme |
| `93-irqbalance.sh` | IRQ affinity helper |
| `94-legacy-aliases.sh` | old-name symlinks for the other board (no-op on R7000) |
| `94-tether.sh` | USB tethering helpers |
| `95-wifi-off.sh` | radios off on first boot |
| `96-tailscale-off.sh` | Tailscale off until a node key exists |
| `97-adguard-off.sh` | AdGuard off until a binary exists in `/opt/adguard` |
| `98-adguard-dns.sh` | move dnsmasq off `:53` when AdGuard is installed |
| `98-ap-mode.sh` | LAN `192.168.1.28/24`, DHCP off, Wi-Fi seed file |
| `99-homelab-enable.sh` | enable only the always-safe services |
| `etc/init.d/router-wifi` | first-boot AP configuration from `/etc/router-wifi.conf` |
| `etc/init.d/adguard-hl` | procd wrapper for `/opt/adguard/AdGuardHome` |
| `etc/init.d/tailscale-hl` | opt-in `tailscaled` (decompressed to `/tmp`) |
| `etc/init.d/router-irq` | IRQ pinning for the network cores |
| `etc/init.d/flash-ml` | mount helper for legacy flash volumes |
| `usr/sbin/wifi-radio` | `on` / `off` / `status` for the APs |
| `www/router.html`, `www/luci-static/resources/view/router.js` | Router project page |
| `www/cgi-bin/router-status`, `wifi-status`, `speed-up`, `speed-down` | small status/speed CGI helpers |
| `usr/share/luci/menu.d/luci-app-router.json` | *Services → ★ Router project* |
| `usr/share/luci/menu.d/luci-app-router-legacy.json` | legacy service path → router |

The LuCI footer badge is **not** here: it lives in `feeds/luci`, so it is
shipped as `patches/0003-luci-footer-badge.patch`.
