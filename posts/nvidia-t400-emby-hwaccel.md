---
title: Adding GPU Hardware Transcoding to a Dell PowerEdge R720 with Emby
date: 2026-04-29
tags: homelab, emby, nvidia, docker, r720, transcoding
---

I've been running Emby on my Dell PowerEdge R720 for a while now. It works great for direct play — 4K Dolby Vision HEVC streams right to the Apple TVs without breaking a sweat. But the moment something needs transcoding, the old Xeon E5-2690s have to do all the work in software, which isn't ideal.

Today I finally fixed that by dropping an Nvidia T400 into the server.

## Why the T400?

The R720 has a power problem for GPUs. It's a 2012-era rack server with no internal GPU power connectors — no 6-pin, no 8-pin. Any card you install has to run entirely off PCIe slot power, which caps you at 75W.

The T400 fits perfectly:
- 70W TDP — runs off the PCIe slot alone
- Full height card — fits the 2U chassis
- NVENC + NVDEC — hardware encode and decode
- Cheap — grabbed one from a Precision workstation I had laying around

## Installing It

The R720 uses riser boards instead of open PCIe slots. The GPU goes into **Riser 2**, which has the full x16 slot. No GPU enablement kit needed for a single-wide card under 75W — just seat it and go.

One note: make sure you have dual CPUs installed. The x16 slot on Riser 2 requires CPU #2 to be active. My server already had dual E5-2690s so no issue there.

## Getting Drivers Working

This is where things got interesting. The server runs OMV (OpenMediaVault) on a custom Debian 12 kernel — `6.12.74+deb12-amd64`. The standard apt driver install works but you need to make sure the kernel headers match:

```bash
sudo apt install -y nvidia-driver nvidia-kernel-dkms linux-headers-$(uname -r)
sudo dkms autoinstall
sudo modprobe nvidia
```

Verify with:

```bash
nvidia-smi
```

You should see the T400 showing up at around 31W cap — that's the PCIe slot power limit being correctly reported.

## Docker and the Nvidia Container Toolkit

Emby runs in Docker, so the GPU needs to be accessible inside the container. Install the Nvidia container toolkit:

```bash
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg
curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
sudo apt update && sudo apt install -y nvidia-container-toolkit
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

## The Missing Libraries

Here's the part that took some digging. Emby uses its own `ffdetect` binary to probe for hardware acceleration, and it needs three specific Nvidia libraries to be visible inside the container:

- `libcuda.so.1`
- `libnvcuvid.so.1`
- `libnvidia-encode.so.1`

These aren't automatically passed through even with `--gpus all`. Install them on the host:

```bash
sudo apt install -y libcuda1 libnvcuvid1 libnvidia-encode1
```

Then mount them into the container explicitly. Here's the full docker run command:

```bash
docker run -d \
  --name embyserver \
  --gpus all \
  --network host \
  --restart unless-stopped \
  -v /data/embyserver/config:/config \
  -v /srv:/media \
  --device /dev/dri:/dev/dri \
  --device /dev/nvidia0:/dev/nvidia0 \
  --device /dev/nvidiactl:/dev/nvidiactl \
  --device /dev/nvidia-uvm:/dev/nvidia-uvm \
  --device /dev/nvidia-uvm-tools:/dev/nvidia-uvm-tools \
  --device /dev/nvidia-caps/nvidia-cap1:/dev/nvidia-caps/nvidia-cap1 \
  --device /dev/nvidia-caps/nvidia-cap2:/dev/nvidia-caps/nvidia-cap2 \
  -v /usr/lib/x86_64-linux-gnu/libcuda.so.1:/usr/lib/x86_64-linux-gnu/libcuda.so.1:ro \
  -v /usr/lib/x86_64-linux-gnu/libnvidia-encode.so.1:/usr/lib/x86_64-linux-gnu/libnvidia-encode.so.1:ro \
  -v /usr/lib/x86_64-linux-gnu/libnvcuvid.so.1:/usr/lib/x86_64-linux-gnu/libnvcuvid.so.1:ro \
  emby/embyserver:latest
```

## Confirming It Works

Test that ffdetect inside the container can see NVENC:

```bash
docker exec embyserver /bin/ffdetect -hide_banner -show_program_version -loglevel 48 -show_error -show_log 40 nvencdec -print_format json 2>&1 | grep -i "loaded\|nvenc\|cuda"
```

You should see:

```
Loaded lib: libcuda.so.1
Loaded lib: libnvcuvid.so.1
Loaded lib: libnvidia-encode.so.1
Loaded Nvenc version 12.1
1 CUDA capable devices found
```

In Emby, go to **Dashboard → Playback → Transcoding**, set hardware acceleration to **Advanced**, and you'll see the T400 listed with NVDEC decoders for H.264, H.265, MPEG-2, VC-1, VP8 and NVENC encoders for H.264 and H.265.

## Real World Results

For my setup — Apple TVs as clients — almost everything direct plays. The Apple TV handles 4K Dolby Vision HEVC natively, so the GPU sits idle most of the time. But it's there when something needs it, and the NVDEC decode offload helps even on mixed transcodes.

The T400 runs cool and quiet in the rack. 35-38°C under load, fan at 35%. For a card running off slot power in a 2012 server, that's a win.

— Jeremiah / K6JRN
