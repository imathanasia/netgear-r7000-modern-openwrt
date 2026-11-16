# Wi-Fi

## Driver

`b43-next` (out-of-tree, from <https://github.com/aleferri/b43-next>) with the
AC-PHY patches from the OpenWrt `b43-ac-phy` tree, firmware squashed to core
revision 42 / AC. Both BCM4360 radios are driven natively — no `wl`, no
`brcmfmac`, no binary blob loaded by the vendor driver.

Observed on this unit:

- `phy0` → 2.4 GHz, HT20, 20 dBm, ESSID `<SSID>-2G`
- `phy1` → 5 GHz, VHT80, 23 dBm, ESSID `<SSID>-5G`
- DFS works with `CONFIG_B43_NEXT_DFS=y` (CAC runs before the AP starts on a
  DFS channel).

## Radios are off by default

Nothing is transmitted until you ask:

```sh
wifi-radio on        # enable + start, persists across reboots
wifi-radio off       # stop + disable
wifi-radio status    # service state, phys, ESSIDs
```

The b43 radios register about 15 s after boot; `router-wifi` waits for both
phys before configuring them. A first `wifi-radio on` right after boot can take
up to ~2 minutes on a cold start.

## Settings

`/etc/router-wifi.conf` (seeded on first boot):

```sh
SSID="R7000"          # -> R7000-2G / R7000-5G
KEY="change-me-now"   # CHANGE THIS
COUNTRY="PL"          # set your regulatory domain
CH_2G="1"
CH_5G="36"            # any DFS channel works, e.g. 52 / 100
HTMODE_5G="VHT80"
```

Apply with:

```sh
/etc/init.d/router-wifi restart
```

`wifi-radio on` enables the service; the script only configures the boards
this overlay supports and returns silently on anything else.

## Practical notes

- 5 GHz VHT80 on ch36 gives the best throughput here; DFS channels add a
  60-second CAC before the AP appears.
- 2.4 GHz is HT20 only in the seeded config. HT40 works but is antisocial in
  dense areas.
- No 160 MHz, no MU-MIMO, no beamforming: those are not implemented by
  b43-next on this hardware.
- Throughput is CPU-bound (Cortex-A9, no hardware NAT). Measure it with
  `iperf3` on your own clients; USB 3.0 storage is CPU-bound too.
- `iw` is not installed; use `iwinfo` (`iwinfo`, `iwinfo <iface> info`).
- The image ships `wifischedule`, so the APs can be time-boxed without
  scripting.
