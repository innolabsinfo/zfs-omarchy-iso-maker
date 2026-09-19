# zfs-mode — drop the ZFS install mode onto a fresh omarchy-iso checkout

Everything the `zfs-install` branch adds, packaged so it can be replayed onto a
clean upstream clone.

## Contents

- `apply-zfs.sh` — self-contained script; the patch is embedded, so this single
  file is all you need. It creates (or switches to) a branch, applies the ZFS
  install mode, commits, and optionally builds the ISO.
- `zfs-mode.patch` — the same change as a plain `git diff`, if you would rather
  apply it yourself.

## Usage

```bash
# 1. original repo, based on the branch the patch was made against (quattro)
git clone https://github.com/omacom/omarchy-iso
cd omarchy-iso
git checkout quattro

# 2. apply the ZFS mode -> creates branch "zfs" and commits
/path/to/zfs-mode/apply-zfs.sh --repo . --branch zfs

# 3. build the ISO (needs Docker + root, exactly like ./bin/omarchy-iso-make)
./bin/omarchy-iso-make --keep-pkg-cache --no-boot-offer
#    -> release/zfs-omarchy-<version>-x86_64-quattro.iso
```

Or apply and build in one go:

```bash
/path/to/zfs-mode/apply-zfs.sh --repo /path/to/omarchy-iso --build
```

Options: `--repo PATH` (default `.`), `--branch NAME` (default `zfs`), `--build`.

## What it changes

ZFS install mode: whole-disk natively-encrypted pool (`aes-256-gcm`,
passphrase), `/boot` inside the encrypted root, ZFSBootMenu on the ESP, no
btrfs / Limine / Snapper. Encrypted-pool passphrase == the user's password.

Files touched (11):

| File | Role |
|---|---|
| `configs/airootfs/usr/share/omarchy-iso/zfs.sh` | new — pool/datasets/initramfs/ZBM helper |
| `configs/airootfs/usr/share/omarchy-iso/orchestrator/{context,main,phases_impl}.py` | `mode: zfs` phases |
| `configs/airootfs/root/configurator` | "ZFS install" picker + amber accent |
| `configs/airootfs/root/.automated_script.sh` | amber VT palette |
| `builder/build-iso.sh` | ZFS from AUR into the live env + offline mirror, cloud-init mask |
| `configs/airootfs/etc/cloud/cloud.cfg.d/99-omarchy-disable.cfg` | cloud-init inert |
| `configs/airootfs/usr/share/omarchy-iso/disk-partitioning.sh` | loop-device partition paths |
| `configs/profiledef.sh` | `iso_name=zfs-omarchy` (`zfs-` filename prefix) |
| `plans/zfs-install.md` | design + handoff notes |

Includes the three hard-won boot fixes: mkinitcpio `ALL_config` left commented
(so `conf.d` drop-ins apply), `zfs-list.cache` primed without the altroot prefix
+ `zfs-zed`/`zfs.target`, and `zfs_force=1` on the ZBM command line.

## Building the ISO (after applying the patch)

The patch does not change *how* the ISO is built, only its contents and name.
From the repo root, on the `zfs` branch:

```bash
./bin/omarchy-iso-make --keep-pkg-cache --no-boot-offer
```

The result is **`release/zfs-omarchy-<version>-x86_64-quattro.iso`** (the `zfs-`
prefix comes from the patch's `iso_name=zfs-omarchy`).

Or apply the patch and build in one step:

```bash
/path/to/zfs-mode/apply-zfs.sh --repo /path/to/omarchy-iso --branch zfs --build
```

### What happens during the build

1. `git submodule update --init --recursive` (fetches `archiso` from GitLab — needs internet),
2. inside the `archlinux:latest` container it builds `perl-boolean`, `zfs-utils`,
   `zfs-dkms` and `zfsbootmenu` from the AUR,
3. it assembles the offline mirror (the packages used for a network-less install)
   and runs `mkarchiso`,
4. the result lands in `release/`.

For us the whole thing took ~6–7 min with a warm package cache. The first build
on a fresh machine is longer (it downloads the Arch packages and compiles
`zfs-dkms`).

### Requirements / notes

- **Docker** — `omarchy-iso-make` uses `sudo docker` itself when you are not in
  the `docker` group (it will ask for a password).
- **Internet** — for the Arch/AUR packages and the `archiso` submodule.
- `--keep-pkg-cache` — skips the interactive `sudo rm -rf /var/cache/pacman/pkg/*`.
- `--no-boot-offer` — does not ask "Boot …?" when it finishes.
- Offline-mirror cache: `~/.cache/omarchy/iso_<mirror>/` (later builds are faster).

## Where the ISO ends up

When the build finishes, the ISO is in **`./omarchy-iso/release/`**, named
**`zfs-omarchy-<version>-x86_64-<ref>.iso`** — for example
`zfs-omarchy-2026.09.19-x86_64-quattro.iso`.
