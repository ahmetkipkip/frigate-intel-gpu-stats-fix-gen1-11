# Frigate Intel GPU Stats Fix

A drop-in wrapper for `intel_gpu_top` that fixes **GPU usage showing 0%** in [Frigate NVR](https://frigate.video/) — specifically for older Intel GPUs (Gen 1–11, e.g., Haswell, Broadwell, Skylake, Kaby Lake) and Proxmox LXC setups where the GPU appears as `card1` instead of `card0`.

> **Note:** If you have an Intel Gen 12+ CPU (Alder Lake N100/N200, Raptor Lake, etc.), check out [sfortis/frigate-intel-gpu-stats](https://github.com/sfortis/frigate-intel-gpu-stats) which uses the DRM fdinfo interface available on those newer processors.

## The Problem

Frigate internally polls GPU stats by running:

```
timeout 0.5s intel_gpu_top -J -o - -s 1000
```

This has **two issues** that cause GPU stats to show 0%:

### 1. Timing Issue (All Intel GPUs)

With `-s 1000` (1-second sample interval), `intel_gpu_top` outputs an **initialization JSON with all zeros** immediately, then real data after 1000ms. But `timeout` kills the process at 500ms — so Frigate **only ever sees zeros**.

```
0ms   → First JSON output (all zeros — initialization)
500ms → timeout kills the process ← Frigate reads this
1000ms → Real data would have arrived here (never reached)
```

### 2. Wrong GPU Card (Proxmox LXC / Multi-GPU)

In Proxmox LXC containers, the GPU often appears as `/dev/dri/card1` instead of `card0`. `intel_gpu_top` defaults to looking for `card0` and silently fails — reporting 0% even though the GPU is actively decoding video.

You can verify this yourself:

```bash
# Inside the Frigate container:
intel_gpu_top -L
# Output: card1  Intel Haswell (Gen7)  pci:vendor=8086,device=0412,card=0

# Stats work when specifying the device manually:
intel_gpu_top -d "pci:card=0" -s 1000 -l 1
# Shows real GPU usage!
```

## The Solution

This wrapper script:

1. **Overrides the sample interval** from 1000ms to 250ms, so real data arrives within the 0.5s timeout
2. **Auto-detects the GPU device**, fixing the `card1` vs `card0` issue
3. Passes all other arguments through to the original `intel_gpu_top` binary

## Installation

### Step 1: Copy the original binary

First, make sure Frigate is running, then copy the original binary out of the container:

```bash
docker cp frigate:/usr/bin/intel_gpu_top /path/to/frigate/config/intel_gpu_top.orig
```

### Step 2: Download the wrapper script

```bash
wget -O /path/to/frigate/config/intel_gpu_top \
  https://raw.githubusercontent.com/YOUR_USERNAME/frigate-intel-gpu-stats-fix/main/intel_gpu_top

chmod +x /path/to/frigate/config/intel_gpu_top
```

### Step 3: Add volume mounts to `docker-compose.yml`

```yaml
services:
  frigate:
    # ... your existing config ...
    volumes:
      # ... your existing volumes ...

      # GPU stats fix: mount wrapper as intel_gpu_top, original as intel_gpu_top.orig
      - /path/to/frigate/config/intel_gpu_top.orig:/usr/bin/intel_gpu_top.orig:ro
      - /path/to/frigate/config/intel_gpu_top:/usr/bin/intel_gpu_top:ro
```

### Step 4: Recreate the container

```bash
docker compose down
docker compose up -d
```

GPU stats should appear in the Frigate UI within 30 seconds.

## Full docker-compose.yml Example

```yaml
services:
  frigate:
    container_name: frigate
    image: ghcr.io/blakeblackshear/frigate:stable
    restart: unless-stopped
    privileged: true
    shm_size: "256mb"
    volumes:
      - /opt/frigate/config:/config
      - /mnt/recordings/frigate:/media/frigate
      - /opt/frigate/config/intel_gpu_top.orig:/usr/bin/intel_gpu_top.orig:ro
      - /opt/frigate/config/intel_gpu_top:/usr/bin/intel_gpu_top:ro
      - type: tmpfs
        target: /tmp/cache
        tmpfs:
          size: 1000000000
    ports:
      - "5000:5000"
      - "8554:8554"
      - "8555:8555/tcp"
      - "8555:8555/udp"
    devices:
      - /dev/dri/card1:/dev/dri/card1
      - /dev/dri/renderD128:/dev/dri/renderD128
    environment:
      LIBVA_DRIVER_NAME: i965  # Required for Gen 1-5 (Haswell, Broadwell, etc.)
```

## Configuration

### GPU Device Override

The wrapper auto-detects the GPU device. If auto-detection doesn't work for your setup, you can manually specify the device via environment variable:

```yaml
    environment:
      GPU_DEVICE: "pci:card=0"  # or "drm:/dev/dri/renderD128"
```

### Intel Driver Selection

Choose the correct driver for your CPU generation:

| CPU Generation | Driver | Environment Variable |
|---|---|---|
| Gen 1–5 (Haswell, Broadwell, etc.) | i965 | `LIBVA_DRIVER_NAME: i965` |
| Gen 6+ (Skylake and newer) | iHD | `LIBVA_DRIVER_NAME: iHD` (default) |

## Proxmox LXC Setup

If you're running Frigate in a Proxmox LXC container, make sure:

### 1. GPU is passed through to the LXC

In `/etc/pve/lxc/<CTID>.conf`:

```
lxc.cgroup2.devices.allow: c 226:0 rwm
lxc.cgroup2.devices.allow: c 226:128 rwm
lxc.mount.entry: /dev/dri dev/dri none bind,optional,create=dir
```

Or use Proxmox GUI: LXC → Resources → Device Passthrough.

### 2. Permissions are set on the host

```bash
# Temporary
chmod 666 /dev/dri/renderD128

# Permanent (survives reboot)
cat > /etc/udev/rules.d/99-gpu-permissions.rules << 'EOF'
KERNEL=="renderD128", MODE="0666"
EOF
```

### 3. perf_event_paranoid is set on the host

```bash
# Temporary
sh -c 'echo 0 > /proc/sys/kernel/perf_event_paranoid'

# Permanent
sh -c 'echo kernel.perf_event_paranoid=0 >> /etc/sysctl.d/local.conf'
```

## Troubleshooting

### Verify VAAPI is working

```bash
docker exec -it -e LIBVA_DRIVER_NAME=i965 frigate vainfo --display drm --device /dev/dri/renderD128
```

You should see supported profiles (H264, MPEG2, etc.).

### Verify GPU is actually being used

```bash
# On Proxmox host:
intel_gpu_top

# Look for VCS/0 (Video) showing > 0%
```

### Check what Frigate sees

```bash
# Simulate exactly what Frigate runs:
docker exec -it frigate bash -c 'timeout 0.5s intel_gpu_top -J -o - -s 1000 2>/dev/null'
```

You should see two JSON blocks — the first with zeros, the second with real values.

### Check the wrapper is running

```bash
docker exec -it frigate intel_gpu_top -L
```

### Common issues

| Symptom | Cause | Fix |
|---|---|---|
| GPU section missing entirely in Frigate UI | `intel_gpu_top` binary not found or crashing | Check volume mounts are correct |
| GPU shows 0% | Timing issue or wrong card | Install this wrapper |
| `vainfo` shows errors | Driver not loaded or wrong driver | Set correct `LIBVA_DRIVER_NAME` |
| No `/dev/dri/` in container | GPU not passed through | Fix LXC config (see Proxmox section) |

## How It Works

```
┌─────────────────────────────────────────────────────┐
│ Frigate                                             │
│  runs: timeout 0.5s intel_gpu_top -J -o - -s 1000  │
└──────────────────────┬──────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────┐
│ Wrapper (this script)                                │
│  1. Intercepts -s 1000 → replaces with -s 250       │
│  2. Auto-detects GPU device (card0/card1/etc.)       │
│  3. Calls original binary with fixed arguments       │
└──────────────────────┬───────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────┐
│ Original intel_gpu_top.orig                          │
│  -d "pci:card=0" -J -o - -s 250                     │
│                                                      │
│  0ms   → JSON #1 (zeros — initialization)            │
│  250ms → JSON #2 (real GPU usage!) ← within timeout  │
│  500ms → timeout kills process (exit 124)            │
└──────────────────────────────────────────────────────┘
```

## Compatibility

- **Tested on:** Intel Core i5-4590 (Haswell, Gen 4) in Proxmox 8.x LXC
- **Should work on:** Any Intel GPU where Frigate shows 0% GPU usage
  - Haswell (4th Gen)
  - Broadwell (5th Gen)
  - Skylake (6th Gen)
  - Kaby Lake (7th Gen)
  - Coffee Lake (8th/9th Gen)
  - Comet Lake (10th Gen)
  - Tiger Lake / Rocket Lake (11th Gen)
- **Frigate versions:** 0.14.x, 0.15.x, 0.16.x, 0.17.x
- **Also fixes:** Multi-GPU setups where `intel_gpu_top` defaults to wrong card

> For Gen 12+ (Alder Lake, Raptor Lake), see [sfortis/frigate-intel-gpu-stats](https://github.com/sfortis/frigate-intel-gpu-stats) which uses a different approach (DRM fdinfo).

## Credits

- Inspired by [sfortis/frigate-intel-gpu-stats](https://github.com/sfortis/frigate-intel-gpu-stats) (Gen 12+ solution using DRM fdinfo)
- Root cause analysis from [Frigate GitHub Discussion #21319](https://github.com/blakeblackshear/frigate/discussions/21319) and [#16619](https://github.com/blakeblackshear/frigate/discussions/16619)

## License

MIT
