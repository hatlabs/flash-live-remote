# flash-live-remote

Flash a Linux SBC over the network while it's running. Streams a disk image from your development machine to a remote device, overwrites its boot device, and reboots into the new image.

## Prerequisites

**Local (dev machine):**
- `ncat` (from nmap: `brew install nmap` on macOS, `apt install ncat` on Debian)
- `ssh`
- `xzcat` / `zcat` (for compressed images)

**Target device:**
- `busybox` (with `nc` applet)
- SSH access
- Linux with `systemd`, `findmnt`, `lsblk`, `dd`

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
flash-live-remote <host> <image>
```

Supported image formats: `.img`, `.img.xz`, `.img.gz`

```bash
# Flash a compressed image
flash-live-remote halos.local ~/images/halos-marine.img.xz

# Flash an uncompressed image
flash-live-remote pi@192.168.1.100 image.img
```

## How it works

```
Dev machine                               Target (Linux SBC)
───────────                               ──────────────────
Phase 0: SSH ──────────────────────────── Resolve IP, check busybox
Phase 1: SSH ──────────────────────────── Detect root block device
         SSH ──────────────────────────── Copy helper script to /dev/shm
         SSH ──────────────────────────── Launch helper (detached)
                                          Helper: stop services, sync
                                          Helper: listen on signal port 9998
Phase 2: ncat ── "GO" / "READY" ──────── Handshake (no destructive action yet)
                                          Helper: listen on data port 9999
         xzcat | ncat --send-only ─────── dd writes to raw block device
                                          Helper: sync, reboot -f
```

### Key design decisions

- **`ncat` (from nmap), not macOS `nc`**: macOS `nc` doesn't close the TCP connection when stdin reaches EOF, causing the transfer to hang indefinitely. `ncat --send-only` closes correctly.
- **Two-port handshake**: A signal port (9998) confirms bidirectional connectivity *before* any destructive action. If the handshake fails, the helper reboots to restore services. The device is always recoverable at this point.
- **`busybox nc` on target**: busybox `nc -l` accepts exactly one connection then exits. Never probe with `nc -z` — the probe consumes the listener.
- **No pivot_root or unmount**: `dd` writes to the raw block device, bypassing the mounted filesystem. `reboot -f` skips sync so the old filesystem cache doesn't write back. Much simpler than unmounting root.
- **Hostname resolved to IP early**: mDNS (`.local`) resolution fails after avahi is stopped. The target's IP is captured via SSH before services are stopped.

## Limitations

- **If the transfer fails after the handshake**, the device has a partial image. It will attempt to reboot but may not come back — manual re-flash (SD card / NVMe adapter) is required.
- The helper script stops `container-*`, `marine-*`, `halos-*`, `cockpit`, and `docker` services. Adjust if your target runs different services.
- Ports 9998 and 9999 are hardcoded.
- Only tested with Raspberry Pi (HaLOS) but should work on any Linux SBC with the listed prerequisites.

## License

MIT
