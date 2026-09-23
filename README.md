# dArkOS RK2023 → PowKiddy X35H / X35S - **AUTOMATED BUILDING - ENGLISH**

Modifies the official **RK2023** image of [dArkOS v08272026](https://github.com/christianhaitian/dArkOS/releases/tag/v08272026) to a version that can boot on the **PowKiddy X35H / X35S**, with automated building, GitHub Release publishing, and Baidu Netdisk uploading via GitHub Actions.

To build a single variant only, execute inside script folder:

**WSL/Linux**

sudo bash build-all.sh --variant X35S --skip-compress

OR

sudo bash build-all.sh --variant X35H --skip-compress



## Modifications

| Component | File | Description |
| --- | --- | --- |
| U-Boot | `RK3566-Specific_uboot.bin` | Written to image sector 64 (consistent with `flash-uboot.sh`) |
| Kernel | `Image` | Replaces `/Image` in the boot partition |
| DTB | `rk3566-powkiddy-x35h.dtb` / `rk3566-powkiddy-x35s.dtb` | Copied to the root directory of the boot partition |
| extlinux | `overlay/extlinux/*.extlinux.conf` | Replaces the entire file; contains three changes in APPEND relative to upstream (see below) |
| rootfs | `overlay/rootfs/` | s2idle, device identifier, audio, and emulator recovery scripts (see below) |

### extlinux.conf APPEND Changes (Relative to Upstream RK2023)

| Item | Upstream | This Mod |
| --- | --- | --- |
| LCD Console | `console=tty1` | **Removed** |
| Kernel Log Level | `loglevel=5` | **Removed** |
| Serial Console | None | `console=ttyS2,1500000n8` (Logs routed via UART, avoiding LCD occupation) |

Additionally, `FDT` is changed to the corresponding DTB for X35H / X35S. Templates can be found under `overlay/extlinux/`.

### rootfs Changes

| Path | Description |
| --- | --- |
| `/etc/systemd/sleep.conf.d/s2idle.conf` | Forces `MemorySleepMode=s2idle` and `SuspendState=mem` (This board cannot wake up from deep sleep) |
| `/home/ark/.config/.DEVICE` | Set to `X35H` / `X35S` (Written during build according to the variant; no longer inherits upstream `RK2023`) |
| `/usr/local/bin/spktoggle.sh` | X35 silence recovery / Speaker path set to **SPK** (RK2023/RGB30 still use historical HP logic) |
| root crontab | **Deletes** `@reboot spktoggle.sh` (If already in SPK mode at boot, this script would toggle it to HP → speaker remains silent) |
| `/usr/local/bin/Fix Audio.sh` | Identifies X35H/X35S and sets **SPK** when repairing audio |
| `/opt/system/Advanced/Fix Audio.sh` | Same as above |
| `/usr/local/bin/headphone-audio-switch.sh` | Reads extcon `HEADPHONE=` first, then falls back to dmesg |
| `Restore Default {Drastic,GZdoom,LZdoom,PPSSPP}*.sh` | X35 uses the RK2023 configuration files (both are 640×480) |

Templates are located in `overlay/rootfs/` and copied over during the build process after mounting the rootfs partition (p4, btrfs); `.DEVICE` is overwritten by `mod-image.sh` based on the specified variant.

> **Note:** Upstream incorrectly sets the "speaker" for RK2023/RGB30 to `Playback Path=HP`. On the X35, HP turns off `spk-ctl`, resulting in no sound from the speaker; **SPK** is required.

The X35H and X35S share the same hardware and differ only in screen orientation, so two separate images are output.

## Directory Structure

```
.
.
├── config.env                 # Upstream version, output naming, variant configs
├── flash-uboot.sh             # U-Boot flashing (shared by local/CI)
├── Image                      # Custom kernel
├── RK3566-Specific_uboot.bin  # Custom U-Boot
├── rk3566-powkiddy-x35h.dtb
├── rk3566-powkiddy-x35s.dtb
├── overlay/
│   ├── extlinux/              # extlinux.conf templates
│   │   ├── X35H.extlinux.conf
│   │   └── X35S.extlinux.conf
│   └── rootfs/                # rootfs file overlays (maintaining path consistency)
│       ├── etc/systemd/sleep.conf.d/s2idle.conf
│       ├── home/ark/.config/.DEVICE
│       ├── usr/local/bin/{spktoggle,Fix Audio,headphone-audio-switch}.sh
│       └── opt/system/Advanced/{Fix Audio,Restore Default *}.sh
└── scripts/
├── download-base.sh       # Download and extract upstream RK2023 image
├── mod-image.sh           # Single-variant mod (uboot + kernel + dtb)
├── build-all.sh           # Build all variants and split with 7z
└── upload-baidu.sh        # Upload to Baidu Netdisk
```

## Local Build

Dependencies: `p7zip-full`, `curl`, `sgdisk` (`gdisk` package), `dosfstools`, `btrfs-progs`.

sudo apt-get install -y p7zip-full curl gdisk dosfstools btrfs-progs
sudo bash scripts/build-all.sh

Build outputs are saved in `dist/` (approx. 3 split volumes per variant, each volume < 2GiB, ready for GitHub Release upload):

- `dArkOS_RK2023_X35H_trixie_06082026.img.7z.001` … `.002` …
- `dArkOS_RK2023_X35S_trixie_06082026.img.7z.001` … `.002` …

The raw `.img` file will be deleted after compression; Release / Baidu Netdisk will only distribute the `.7z.*` split archives.

To build a single variant only:

sudo bash build-all.sh --variant X35S --skip-compress
OR
sudo bash build-all.sh --variant X35H --skip-compress

## Flashing U-Boot Manually (Optional)

If you already have a dArkOS RK2023 image or SD card, you can flash U-Boot separately:

sudo ./flash-uboot.sh /dev/sdX          # SD Card
sudo ./flash-uboot.sh darkos-rk2023.img  # Image file

## GitHub Actions

Workflow file: `.github/workflows/build-release.yml`

### Trigger Methods

1. **Tag & Push** (Recommended)  
   git tag v1.0.0
   git push origin v1.0.0
2. **Manual Execution**: Actions → Build and Release → Run workflow

### Actions Artifacts (Manual Build & Download)

`upload-artifact` will **zip the files again** before uploading to Actions; every upload step on the page equals **1 artifact** entry. This is standard GitHub behavior and does not mean the split archive failed.

Download and extract the artifact zip file to access the underlying `.7z.001`, `.7z.002`, etc., split archives.

| Artifact Name | Content |
| --- | --- |
| `darkos-x35h-7z` | All X35H split volumes |
| `darkos-x35s-7z` | All X35S split volumes |
| `release-notes` | Release Notice |

If you want each volume **listed separately** for direct download, create a tag and publish a **GitHub Release** (single file attachment < 2GiB).

### Repository Secrets (Baidu Netdisk)

| Secret | Description |
| --- | --- |
| `BDUSS` | Baidu account BDUSS Cookie |
| `STOKEN` | Baidu account STOKEN Cookie |

Obtain these from your browser cookies after logging in at [pan.baidu.com](https://pan.baidu.com) (Do not leak these).

### Repository Variables (Optional)

| Variable | Default Value | Description |
| --- | --- | --- |
| `BAIDU_REMOTE_DIR` | `/Apps/dArkOS-X35/` | Root directory on Baidu Netdisk (automatically creates date subdirectories underneath) |

Actual upload path: `/Apps/dArkOS-X35/YYYY-MM-DD/` (Date set to `Asia/Shanghai`).  
Optional variable `BAIDU_DATE` overrides the date folder; locally, you can specify `BAIDU_DATE=2026-07-19`.
| `DARKOS_RELEASE` | `v06072026` | Upstream release version referenced in the Release Notes |

If `BDUSS` / `STOKEN` are not configured, builds and GitHub Releases will still execute, skipping only the Baidu upload step.

## Updating Mod Resources

1. Replace `Image`, `RK3566-Specific_uboot.bin`, or DTB files in the repository root directory.
2. If the upstream dArkOS version changes, update `DARKOS_RELEASE` and `BASE_IMAGE_BASENAME` in `config.env`.
3. Push a new tag to trigger CI, or run `sudo bash scripts/build-all.sh` locally to verify.

## Flashing Instructions

1. Download all `.7z.00x` volumes for your target variant (`x35h` / `x35s`).
2. Open `.001` with 7-Zip and extract the `.img` file.
3. Write to an SD card using Rufus, balenaEtcher, `dd`, etc.

## Licenses and Disclaimers

- Update dArkOS to POWKIDDY X35S/X35H, thank you to [AveyondFly](https://github-com.translate.goog/AveyondFly/darkos_x35)
- Upstream dArkOS copyright belongs to [christianhaitian/dArkOS](https://github.com/christianhaitian/dArkOS).
- This repository provides only image modding scripts and automation workflows; perform at your own risk. Please back up your data first.
