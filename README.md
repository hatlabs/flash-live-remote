# flash-live-remote

Flash a Linux SBC over the network while it's running. Transfers a disk image from your development machine to a remote device, overwrites its boot device, and reboots into the new image.

## Prerequisites

**Local (dev machine):**
- `ssh`, `scp`
- `xzcat` / `zcat` (for compressed images)

**Target device:**
- `busybox`
- SSH access
- Linux with `systemd`, `findmnt`, `lsblk`, `dd`
- Ramfs mode: enough RAM in `/dev/shm` to hold the compressed image (~4 GB+ devices)
- Ramfs mode: `xzcat` / `zcat` (for compressed images — present by default on Raspberry Pi OS)

## Installation

```bash
# Clone and symlink
git clone https://github.com/hatlabs/flash-live-remote.git
ln -s "$PWD/flash-live-remote/flash-live-remote" /usr/local/bin/

# Or just download the script
curl -o /usr/local/bin/flash-live-remote \
  https://raw.githubusercontent.com/hatlabs/flash-live-remote/main/flash-live-remote
chmod +x /usr/local/bin/flash-live-remote
```

## Usage

```bash
flash-live-remote [--ramfs|--stream] <host> <image>
```

Supported image formats: `.img`, `.img.xz`, `.img.gz`

### Ramfs mode (default)

Copies the compressed image to the target's `/dev/shm` via scp, then decompresses and writes it locally. This is the safest mode — if the transfer fails, no destructive action has happened.

```bash
# Default ramfs mode
flash-live-remote halos.local ~/images/halos-marine.img.xz

# Explicit ramfs mode
flash-live-remote --ramfs halos.local ~/images/halos-marine.img.xz
```

### Stream mode

Decompresses locally and streams the uncompressed image over SSH. Uses no extra RAM on the target but has no pre-flash integrity check — if the transfer fails mid-stream, the device has a partial image.

```bash
flash-live-remote --stream pi@192.168.1.100 image.img.xz
```

## How it works

### Ramfs mode

```
Dev machine                               Target (Linux SBC)
───────────                               ──────────────────
Phase 0: SSH ──────────────────────────── Detect root block device
         SSH ──────────────────────────── Check /dev/shm capacity, xzcat/zcat
         Confirm with user
Phase 1: scp image.img.xz ─────────────── /dev/shm/image.img.xz
         SSH ──────────────────────────── Verify file size matches
Phase 2: SSH ──────────────────────────── Stop services, sync
         SSH ──────────────────────────── Deploy helper to /dev/shm
         SSH ──────────────────────────── Launch helper (detached)
                                          Helper: xzcat /dev/shm/image | dd
                                          Helper: rm image, reboot -f
```

### Stream mode

```
Dev machine                               Target (Linux SBC)
───────────                               ──────────────────
Phase 0: SSH ──────────────────────────── Detect root block device
         Confirm with user
Phase 1: SSH ──────────────────────────── Deploy helper to /dev/shm
         SSH ──────────────────────────── Stop services, sync
Phase 2: xzcat | ssh "dd of=/dev/..." ──── dd writes to block device
                                          Helper: reboot -f (via EXIT trap)
```

### Key design decisions

- **Ramfs mode is default**: scp provides a progress bar and pre-flash integrity check (file size verification). If the transfer fails, no destructive action has happened — the device is fully recoverable.
- **Stream mode uses SSH as transport**: The decompressed image is piped directly through SSH (`xzcat | ssh host "dd ..."`). SSH handles flow control, buffering, and connection lifecycle correctly. When xzcat finishes, SSH sends EOF, dd gets EOF and exits. No nc/ncat needed.
- **No pivot_root or unmount**: `dd` writes to the raw block device, bypassing the mounted filesystem. `reboot -f` skips sync so the old filesystem cache doesn't write back.
- **Resolved binary paths**: After dd overwrites the disk, PATH lookups against the rootfs may fail (page cache eviction). All helpers resolve binary paths (busybox, dd) before dd starts.
- **EXIT trap (stream mode)**: The helper uses `trap 'busybox reboot -f' EXIT` to ensure reboot happens even if the SSH session drops during or after the write.

## Choosing a mode

| | Ramfs (default) | Stream |
|---|---|---|
| **Safety** | Pre-flash integrity check; no destructive action until verified | Transfer failure = partial image |
| **Progress** | scp progress bar | None (dd reports at end) |
| **RAM required** | Compressed image must fit in /dev/shm | Minimal |
| **Speed** | Fast (local decompression + NVMe write) | Slower (network-bound for uncompressed data) |
| **Local tools** | ssh, scp | ssh, xzcat/zcat |
| **Target tools** | xzcat/zcat, busybox | busybox |

Use `--stream` for devices with less than ~4 GB RAM where the compressed image won't fit in `/dev/shm`.

## Limitations

- **Stream mode**: If the transfer fails mid-stream, the device has a partial image and may need manual re-flash.
- The helper script stops `container-*`, `marine-*`, `halos-*`, `cockpit`, and `docker` services. Adjust if your target runs different services.
- Only tested with Raspberry Pi (HaLOS) but should work on any Linux SBC with the listed prerequisites.

## License

MIT
