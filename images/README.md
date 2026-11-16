# Images

All screenshots were captured from the running R7000 with headless Chromium (1440×900, 2×
scale); MAC addresses are redacted. A page that returned 404 or the login form failed the
capture, so a broken path could not silently ship a blank screenshot.

## Banner and badges

| File | Use |
|---|---|
| `logo-netgear-r7000.svg` | README hero, 980×300 |
| `badges/openwrt.svg`, `kernel.svg`, `soc.svg`, `wifi.svg`, `ram.svg`, `flash.svg` | README badge row |
| `social-preview.png` | 1280×640 GitHub *Settings → Social preview* (upload manually) and link previews |
| `collage.png` | 1600×1000 six-shot grid used in the README |

## Individual screenshots

| File | Page |
|---|---|
| `luci-login.png` | LuCI login |
| `luci-overview.png` | Status → Overview |
| `luci-wireless.png` | Network → Wireless (radios on, both APs up) |
| `luci-network.png` | Network → Interfaces |
| `luci-sqm.png` | Services → SQM |
| `luci-ksmbd.png` | Services → Network Shares (SMB3) |
| `luci-nlbwmon.png` | Statistics → nlbwmon |
| `luci-vnstat.png` | Statistics → vnstat2 |
| `luci-aria2.png` | Services → aria2 |
| `luci-banip.png` | Services → banip |
| `luci-system.png` | System → System |
| `luci-ttyd.png` | Services → ttyd web terminal |
| `router-page.png` | Services → ★ Router project (viewport) |
| `router-page-full.png` | Services → ★ Router project (full page) |
| `adguard-login.png` | AdGuard Home first-run |
| `adguard-dashboard.png` | AdGuard Home dashboard |

Every shot except `luci-login.png` is taken after logging in; the capture script fails a page
that returns 404 or the login form, so a broken path cannot silently ship a blank screenshot.
