# flash-live-system

Flash a Linux SBC — locally on the device itself or over the network. Transfers a disk image, overwrites the boot device, and reboots into the new image.

## Prerequisites

**Local mode (default — on the device itself):**
- Root access (run with `sudo`)
- `busybox`, `findmnt`, `lsblk`, `dd`, `xzcat` / `zcat`
- Ramfs mode: enough RAM in `/dev/shm` to hold the compressed image
- Stream mode: image must be on a different block device than the one being flashed

**Remote mode (dev machine):**
- `ssh`, `scp`
- `xzcat` / `zcat` (for compressed images, stream mode only)
- `openssl` (only if using `password` in config)
- `python3` (only if using `wifi-ssid` in config)

**Remote mode (target device):**
- `busybox` (with `fdisk`, `losetup`, `mount`, `umount` applets if using customization)
- SSH access
- Linux with `systemd`, `findmnt`, `lsblk`, `dd`
- Ramfs mode: enough RAM in `/dev/shm` to hold the compressed image (~4 GB+ devices)
- Ramfs mode: `xzcat` / `zcat` (for compressed images — present by default on Raspberry Pi OS)

## Installation

```bash
# Clone and symlink
git clone https://github.com/hatlabs/flash-live-system.git
ln -s "$PWD/flash-live-system/flash-live-system" /usr/local/bin/

# Or just download the script
curl -o /usr/local/bin/flash-live-system \
  https://raw.githubusercontent.com/hatlabs/flash-live-system/main/flash-live-system
chmod +x /usr/local/bin/flash-live-system
```

## Usage

```bash
# Local (default — on the device itself, auto-detects best mode)
sudo flash-live-system [--ramfs|--stream] [--config FILE] <image>

# Remote (over SSH)
flash-live-system --remote <host> [--ramfs|--stream] [--config FILE] <image>
```

Supported image formats: `.img`, `.img.xz`, `.img.gz`

### First-boot customization

Use `--config FILE` to pre-configure the flashed image via cloud-init. The config file uses simple `key=value` format — see `example.conf` for all available keys.

```bash
# Flash with customization
sudo flash-live-system --config myboat.conf image.img.xz

# Remote with customization
flash-live-system --remote halos.local --config myboat.conf image.img.xz
```

Config file format:

```ini
# myboat.conf
hostname=myboat
user=mairas
password=secret123
ssh-key=~/.ssh/id_ed25519.pub
ssh-key=~/.ssh/id_rsa.pub
wifi-ssid=MyNetwork
wifi-password=MyPass
wifi-country=FI
```

| Key | Description |
|---|---|
| `hostname` | Set system hostname |
| `user` | Set username (replaces default 'pi') |
| `password` | Set user password (hashed locally with `openssl passwd -6`) |
| `ssh-key` | SSH public key file (repeatable; defaults to `~/.ssh/*.pub` if `user` is set) |
| `wifi-ssid` | WiFi network name (requires `wifi-password`) |
| `wifi-password` | WiFi password (requires `wifi-ssid`) |
| `wifi-country` | WiFi regulatory domain (default: `GB`) |

Tilde (`~/`) in `ssh-key` paths is expanded automatically. The `ssh-key` key can appear multiple times to add multiple keys.

After `dd` completes, the helper mounts the boot partition (FAT32, partition 1) and writes `user-data` and/or `network-config` files for cloud-init. If no config file is given, behavior is identical to a plain flash.

**Important:** Customization assumes the new image has a FAT32 boot partition as partition 1. After `dd`, the helper parses the new partition table via `busybox fdisk` and loop-mounts at the correct offset, so the new image's partition layout does not need to match the old one. However, if the new image lacks a FAT32 boot partition, customization will fail (the flash itself still succeeds and the device reboots normally).

### Local mode (default)

Runs directly on the device being flashed, without SSH. The script **auto-detects** the best flash method:
1. If `/dev/shm` has enough space for the image → uses **ramfs** (safe for same-disk images)
2. If `/dev/shm` is too small but the image is on a **different disk** → uses **stream**
3. If `/dev/shm` is too small and the image is on the **same disk** → fails with an error

You can override auto-detection with `--ramfs` or `--stream`.

```bash
# Auto-detect best mode (recommended)
sudo flash-live-system image.img.xz

# Force ramfs (e.g., you know there's enough /dev/shm)
sudo flash-live-system --ramfs image.img.xz

# Force stream from USB drive
sudo flash-live-system --stream /mnt/usb/image.img.xz

# With first-boot customization
sudo flash-live-system --config myboat.conf image.img.xz
```

### Remote mode

Use `--remote <host>` to flash a device over SSH. Defaults to ramfs mode.

```bash
# Remote ramfs (default)
flash-live-system --remote halos.local ~/images/halos-marine.img.xz

# Remote stream
flash-live-system --remote pi@192.168.1.100 --stream image.img.xz
```

### Ramfs mode (default)

Copies the compressed image to the target's `/dev/shm` (via `cp` locally, or scp remotely), then decompresses and writes it. This is the safest mode — if the transfer fails, no destructive action has happened.

### Stream mode

Decompresses and pipes the image directly to `dd`. In remote mode, the stream goes over SSH. In local mode, the image must be on a different block device (e.g., USB stick) — a safety check prevents streaming from the disk being flashed.

## How it works

### Local ramfs mode

```
On the device itself:
──────────────────────
Phase 0: Detect root block device
         Check /dev/shm capacity, decompression tools
         Check customization prerequisites*
         Confirm with user
         Write cloud-init payloads to /dev/shm*
Phase 1: cp image.img.xz → /dev/shm/image.img.xz
         Verify file size matches
Phase 2: Stop services, sync
         Deploy helper (copies busybox to tmpfs)
         Helper: xzcat /dev/shm/image | dd
         Helper: apply-customization.sh*
         Helper: reboot -f
```
*Only when --config is provided.

### Local stream mode

```
On the device itself:
──────────────────────
Phase 0: Detect root block device
         Check customization prerequisites*
         Confirm with user
         Write cloud-init payloads to /dev/shm*
Phase 1: Safety check (image must be on different block device)
         Stop services, sync
         Copy binaries (busybox, dd, decompressor) to /dev/shm
Phase 2: /dev/shm/decompress image | /dev/shm/dd → block device
         apply-customization.sh*
         reboot -f
```
*Only when --config is provided.

### Remote ramfs mode

```
Dev machine                               Target (Linux SBC)
───────────                               ──────────────────
Phase 0: SSH ──────────────────────────── Detect root block device
         SSH ──────────────────────────── Check /dev/shm capacity, xzcat/zcat
         SSH ──────────────────────────── Check customization prerequisites*
         Confirm with user
         SSH ──────────────────────────── Transfer cloud-init payloads*
Phase 1: scp image.img.xz ─────────────── /dev/shm/image.img.xz
         SSH ──────────────────────────── Verify file size matches
Phase 2: SSH ──────────────────────────── Stop services, sync
         SSH ──────────────────────────── Deploy helper (copies busybox to tmpfs)
         SSH ──────────────────────────── Launch helper (detached)
                                          Helper: xzcat /dev/shm/image | dd
                                          Helper: loop-mount boot partition*
                                          Helper: write cloud-init files*
                                          Helper: reboot -f
```
*Only when --config is provided.

### Remote stream mode

```
Dev machine                               Target (Linux SBC)
───────────                               ──────────────────
Phase 0: SSH ──────────────────────────── Detect root block device
         SSH ──────────────────────────── Check customization prerequisites*
         Confirm with user
         SSH ──────────────────────────── Transfer cloud-init payloads*
Phase 1: SSH ──────────────────────────── Deploy helper (copies busybox to tmpfs)
         SSH ──────────────────────────── Stop services, sync
Phase 2: xzcat | ssh "helper" ─────────── dd writes to block device
                                          Helper: loop-mount boot partition*
                                          Helper: write cloud-init files*
                                          Helper: reboot -f (via EXIT trap)
```
*Only when --config is provided.

### Key design decisions

- **Ramfs mode is default**: scp/cp provides a pre-flash integrity check (file size verification). If the transfer fails, no destructive action has happened — the device is fully recoverable.
- **Stream mode uses SSH as transport** (remote): The decompressed image is piped directly through SSH (`xzcat | ssh host "dd ..."`). SSH handles flow control, buffering, and connection lifecycle correctly. When xzcat finishes, SSH sends EOF, dd gets EOF and exits. No nc/ncat needed.
- **No pivot_root or unmount**: `dd` writes to the raw block device, bypassing the mounted filesystem. `reboot -f` skips sync so the old filesystem cache doesn't write back.
- **Resolved binary paths**: After dd overwrites the disk, PATH lookups against the rootfs may fail (page cache eviction). All helpers resolve binary paths (busybox, dd) before dd starts. Local stream mode additionally copies the decompressor to tmpfs since it runs concurrently with dd in the same pipe.
- **EXIT trap (stream mode)**: The helper uses `trap 'busybox reboot -f' EXIT` to ensure reboot happens even if the SSH session drops during or after the write.
- **Config file over flags**: Passwords stay in a file (not visible in `ps` or shell history), and settings are written once and reused across flashes.

## Choosing a mode

| | Local ramfs | Local stream | Remote ramfs (default) | Remote stream |
|---|---|---|---|---|
| **Safety** | Pre-flash integrity check | Requires image on separate disk | Pre-flash integrity check | Transfer failure = partial image |
| **Progress** | cp progress (if terminal) | None (dd reports at end) | scp progress bar | None (dd reports at end) |
| **RAM required** | Compressed image in /dev/shm | Minimal | Compressed image in /dev/shm | Minimal |
| **Speed** | Fast | Fast (local I/O only) | Fast (local decompress + NVMe) | Network-bound |
| **Needs SSH** | No | No | Yes | Yes |
| **Needs root** | Yes | Yes | No (sudo on target) | No (sudo on target) |

In local mode (default), the script auto-detects the best mode, so you typically don't need to specify `--ramfs` or `--stream`. Use `--remote <host>` for devices you can reach over SSH — use `--stream` for remote devices with less than ~4 GB RAM where the compressed image won't fit in `/dev/shm`.

## Limitations

- **Stream mode (remote)**: If the transfer fails mid-stream, the device has a partial image and may need manual re-flash.
- **Local stream mode**: The image file must be on a different block device than the one being flashed. Use local ramfs mode if the image is on the same disk.
- The helper script stops `container-*`, `marine-*`, `halos-*`, `cockpit`, and `docker` services. Adjust if your target runs different services.
- Only tested with Raspberry Pi (HaLOS) but should work on any Linux SBC with the listed prerequisites.

## License

MIT
