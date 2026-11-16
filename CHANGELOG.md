# Changelog

## 1.0.1 — 2026-11-16

Router page and release refresh; same firmware base as 1.0.0.

- Removed the unverified claim that the stock updater needs `Expect:` stripped; the recovery
  card now points at `docs/install.md`.
- Added a **Project docs** card to the router page (README, install, recovery, use cases,
  services).
- Rebuilt and re-verified on hardware: `.chk` `eccad30d…`, `-initramfs.trx` `b5532d34…`;
  page audit reports the badge present, no other-board branding, no MAC text, gates off.
- README/asset refresh from the same build: hero, badges, collage, social preview.

## 1.0.0 — 2026-11-16

Initial public release for the Netgear R7000.

Image

- OpenWrt 25.12 (`r0-25058cf`) on Linux 6.18.60, b43-ac-phy base.
- b43-next AC-PHY driver on both BCM4360 radios; VHT80 and DFS.
- Full 122 MB UBI reclaimed into one `rootfs_data` volume (87.6 MB `/overlay`)
  via the bcm47xxpart ML-writable patch.
- Radios off on first boot; `wifi-radio on|off|status`.
- Default LAN `192.168.1.28/24`, DHCP server off, root `changeme`.
- Router project LuCI page and footer badge.
- Optional AdGuard Home / Tailscale gates so neither starts uninvited.

Included

- Flashable `-squashfs.chk` and a matching `-initramfs.trx` for TFTP netboot.
- Package manifest and `SHA256SUMS`.
- Importable `.config` plus the minimal diffconfig.
- Patches for bcm53xx, the R7000 image definition and the LuCI badge.
- Provenance-pinned `package/kernel/b43-next`.
- Docs: install, recovery, firmware layout, Wi-Fi, services, build,
  limitations, discoveries; 16 screenshots.

Fixed during bring-up (kept here as release notes)

- AdGuard Home no longer auto-starts from the image's rc.d: the build
  auto-enables every init script, so `97-adguard-off.sh` now gates it, exactly
  like Wi-Fi and Tailscale.
- LuCI page no longer points at a `/root/backups` directory that a fresh image
  does not contain; it now points at `mtd0`/`mtd1` backup instructions.
- Neutral footer badge id (`router-badge`), no board-specific DOM ids in the
  rendered page.
- Comment/branding cleanup in the shared overlay so R7000 images name the
  correct hardware.
- `speed-down` capped at 64 MiB and gated off unless a test flag exists.
