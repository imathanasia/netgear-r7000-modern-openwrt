# Services

Everything in the image is stock OpenWrt; AdGuard Home and Tailscale are
deliberately **not** baked in (size, licence, and their own release cadence).

## Soft-float, always

The BCM4709A0 is a Cortex-A9 **without VFP**. Go and other compiled binaries
must be built for soft-float (`GOARM=5`). A hard-float `armv7` binary dies with
`Illegal instruction` on this CPU. Check before copying anything in:

```sh
readelf -A ./binary | grep -i vfp    # should print nothing
```

## AdGuard Home

Use the **armv5 / soft-float** release. Install on the overlay:

```sh
mkdir -p /opt/adguard
# copy the AdGuardHome binary to /opt/adguard/AdGuardHome
chmod 755 /opt/adguard/AdGuardHome

# dnsmasq must release :53 first
uci set dhcp.@dnsmasq[0].port='0'
uci commit dhcp
/etc/init.d/dnsmasq restart

# first run: setup wizard on http://192.168.1.28:3000
/opt/adguard/AdGuardHome -c /opt/adguard/AdGuardHome.yaml --work-dir /opt/adguard
```

Complete the wizard, then enable the service:

```sh
/etc/init.d/adguard-hl enable
/etc/init.d/adguard-hl start
```

The initscript runs `AdGuardHome --no-check-update -c /opt/adguard/AdGuardHome.yaml
--work-dir /opt/adguard`, restarts it on failure, and logs a hint if dnsmasq
still owns `:53`. On a fresh image everything under `/opt` is empty, so nothing
starts until you do the above.

Notes:

- `98-adguard-dns.sh` sets the dnsmasq port automatically **only if** a binary
  already exists at first boot; if you install later, run the `uci` block.
- The UI listens on `:3000`, DNS on `:53` (TCP+UDP, IPv4+IPv6).
- Upstreams, DoH/DoT and filter lists need working internet; in a lab with no
  uplink the UI works but queries time out. That is expected, not a fault.
- Data lives in `/opt/adguard/data` on the overlay — it survives a plain
  `sysupgrade`, and disappears with `-n`.

## Tailscale

The image ships the opt-in plumbing (`/etc/init.d/tailscale-hl`, disabled by
default and gated by `/opt/tailscale/enabled`). Use the **arm (soft-float)**
static build:

```sh
mkdir -p /opt/tailscale
gzip -9 -c tailscaled > /opt/tailscale/tailscaled.gz   # ~28 MB -> decompressed in /tmp at start
cp tailscale /opt/tailscale/tailscale && chmod 755 /opt/tailscale/tailscale

touch /opt/tailscale/enabled
/etc/init.d/tailscale-hl enable
/etc/init.d/tailscale-hl start
/opt/tailscale/tailscale --socket=/tmp/tailscaled.sock up
```

`tailscaled` is decompressed to `/tmp` to keep the overlay small; state is kept
in `/opt/tailscale/state`.

## DNS chain

Default image: clients → dnsmasq(`:53`) → upstream (`192.168.1.1`).

With AdGuard Home: clients → AdGuard(`:53`) → your upstreams, and dnsmasq is
moved to port `0`. The generated AP config keeps the router itself resolving
through the LAN address, so LuCI/opkg keep working once an uplink exists.

## Other services

| Package | Where | Notes |
|---|---|---|
| `ksmbd-server` + `wsdd2` | *Services → Ksmbd* | SMB3 shares, discovered by Windows |
| `aria2` | *Services → Aria2* | download manager, RPC on `6800` |
| `ttyd` | *Services → Terminal* | web shell on `7681` |
| `sqm-scripts` | *Network → SQM* | CAKE/fq_codel shaping |
| `nlbwmon`, `vnstat2` | *Statistics* | per-host and per-interface accounting |
| `banip` | *Services → BanIP* | country/ASN blocklists |
| `wifischedule` | *Network → Wi-Fi schedule* | radio on/off windows |
| `watchcat`, `wol`, `ddns-scripts` | *Services* | watchdog, Wake-on-LAN, dynamic DNS |

The Router project page (*Services → ★ Router project*, `/router.html`) shows
status, storage, CFE notes and these hints in one place.
