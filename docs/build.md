# Build

## Host

Debian/Ubuntu x86_64, ~40 GB free, 4+ GB RAM. Install the standard OpenWrt
build dependencies:

```sh
sudo apt install build-essential clang flex bison g++ gawk gcc-multilib \
  g++-multilib gettext git libncurses-dev libssl-dev python3-distutils \
  python3-setuptools rsync swig unzip zlib1g-dev file wget
```

Run the build as a normal user. OpenWrt's `unpack.mk` can fail on ownership
when run as root; the build tree used here carries a local
`--no-same-owner` workaround, which is **not** shipped — build as non-root
instead.

## Base tree

```sh
git clone https://git.openwrt.org/openwrt/openwrt.git
cd openwrt
git checkout 25058cf725f1994e5d5b4f012d321ecec5ef413d
./scripts/feeds update -a
./scripts/feeds install -a
```

That commit is on the `b43-ac-phy` branch:
"mac80211: b43: add AC-PHY support from b43-ac-wip". Feeds are not pinned
upstream; a much later snapshot may need small fixes.

## Apply this repo

```sh
# 1. LuCI footer badge (applies inside feeds/luci)
git -C feeds/luci apply /path/to/repo/patches/0003-luci-footer-badge.patch

# 2. bcm47xxpart: allow UBI to use the full ML area
cp /path/to/repo/patches/0001-mtd-bcm47xxpart-make-ML-writable.patch \
   target/linux/bcm53xx/patches-6.18/9xx-mtd-bcm47xxpart-make-ML-writable.patch

# 3. image definition: add kmod-b43-next to the R7000 device packages
patch -p1 < /path/to/repo/patches/0002-bcm53xx-netgear-r7000-b43-next.patch

# 4. b43-next package (pinned upstream commit)
mkdir -p package/kernel/b43-next
cp -a /path/to/repo/package/b43-next/. package/kernel/b43-next/

# 5. overlay (network defaults, service gates, LuCI page, branding)
cp -a /path/to/repo/overlay/. files/

# 6. configuration
cp /path/to/repo/config/config.r7000 .config
```

`config/diffconfig.r7000` is the minimal delta over the target defaults; use
`cp config/diffconfig.r7000 .config && make defconfig` if you prefer a smaller
file over `config/config.r7000`.

## Build

```sh
make defconfig
make -j"$(nproc)" download
make -j"$(nproc)"
```

Outputs:

```
bin/targets/bcm53xx/generic/openwrt-bcm53xx-generic-netgear_r7000-squashfs.chk
bin/targets/bcm53xx/generic/openwrt-bcm53xx-generic-netgear_r7000.manifest
```

For a netboot/rescue image as well, enable it and rebuild:

```sh
sed -i 's/^# CONFIG_TARGET_ROOTFS_INITRAMFS is not set$/CONFIG_TARGET_ROOTFS_INITRAMFS=y/' .config
make -j"$(nproc)"       # adds ...-initramfs.trx (the .chk stays identical)
```

## Reproducibility notes

- The kernel, `b43-next` commit and the base tree are pinned; feeds are not.
- `b43-next` is pinned at `ee7ed443dc8dde43bd2b2e9aea7d170ad5e92e1b`
  (GPL-2.0-only) and squashes firmware `6.30.163.46` (784.2) to core rev 42 /
  AC.
- The image build enables every `rc.common` init script (`prepare_rootfs` in
  `include/rootfs.mk`). Optional services must therefore disable themselves in
  `/etc/uci-defaults` on first boot — that is what `95-wifi-off.sh`,
  `96-tailscale-off.sh` and `97-adguard-off.sh` do.
- If you change the overlay's init scripts, also clear stale
  `etc/rc.d` entries from `build_dir/target-*/rootfs-cache-*/root` and
  `build_dir/target-*/root-bcm53xx`; the staging cache otherwise keeps
  symlinks created by an earlier build.
