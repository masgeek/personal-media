# Transcode

Library optimisation: converts video files in place to a single uniform format.
Runs [Unmanic](https://unmanic.app), a library optimiser built around FFmpeg.

The stack is named generically so the backing application can be swapped later
without renaming the service, the volume paths, or the documentation.

## Endpoint

| Item | Value |
|------|-------|
| Web UI port | 8888 |
| Network | `dokploy-network` |
| Upstream address | `transcode:8888` |
| Reachable at | `http://<host-ip>:8888` |

This is an internal service and is not in the root `Caddyfile`. Host port 8888 is
published so the UI is reachable from the LAN. Do not expose it publicly.

## Why No Path Translators

The stack does **not** use host networking and does **not** import libraries from
Radarr or Sonarr. It is pointed straight at the media directories, which is why
no path translator configuration is needed.

This is the main reason it was chosen over Tdarr. Tdarr imports library paths
verbatim from Radarr and Sonarr, which report `D:\Entertainment\...`, so those
paths had to be translated inside the container or every library silently
imported zero files.

## Volumes

| Host path | Container path | Purpose |
|-----------|----------------|---------|
| `../files/transcode/config` | `/config` | settings, presets, installed plugins |
| `../files/transcode/cache` | `/tmp/unmanic` | temporary conversion files |
| `/mnt/d/Entertainment/Movies` | `/library/Movies` | read/write, converts in place |
| `/mnt/d/Entertainment/TV` | `/library/TV` | read/write, converts in place |

All `../files/` paths resolve to `stacks/files/`, consistent with the rest of the
repository. They are bind mounts rather than named volumes so settings and
presets are directly readable and editable on the host.

Media mounts are read/write on purpose. Unmanic converts into the cache directory
and then replaces the original file.

`/mnt/d/Entertainment` is the WSL view of the Windows `D:` drive, which is what
the Docker daemon sees when Dokploy runs inside WSL2.

The cache holds full-size temporary files for every job. Keep
`stacks/files/transcode/cache` on local SSD rather than the media drive.

## First Run

1. Open `http://<host-ip>:8888`.
2. Add the library locations `/library/Movies` and `/library/TV`. These are the
   paths inside the container, not the Windows host paths.
3. Create a conversion preset. For CPU transcoding, use an `libx265` or
   `libx264` encoder.
4. Assign the preset to the library.
5. Use **Run library optimisation** on a small library first and confirm the
   output plays in Jellyfin before enabling the scheduler.

There is no node or worker to start separately. Unmanic processes the queue
itself.

## Walkthrough: Adding a Library and Converting

1. **Libraries → Add library**, choose the location, and paste `/library/Movies`
   or `/library/TV`.
2. Confirm the scan finds files. An empty result means the media mount did not
   resolve; check with `docker exec transcode ls -ld /library/Movies /library/TV`.
3. **Settings → Presets → Create preset.** Pick the container format and video
   encoder.
4. **Settings → Library Optimisation → Assign preset** to the library.
5. Set a schedule, or trigger a manual run.
6. Watch progress from the dashboard. Each conversion writes a full-size
   temporary file into the cache first, so disk usage spikes during a run.

## Hardware Transcoding

**Current state: CPU only.** GPU passthrough is commented out because the NVIDIA
container toolkit is not installed in the WSL2 distribution running Dokploy.

CPU transcoding works out of the box with the bundled FFmpeg. Set presets to a
software encoder such as `libx265`, and expect long job times.

Your RTX 4050's NVENC encoder is substantially faster than software encoding, so
enabling the GPU is worth doing when convenient.

## Enabling NVIDIA Later

The GPU is already visible to WSL2. Confirm that first:

```bash
nvidia-smi -L
```

If it lists the card, the only remaining problem is that Docker cannot pass the
device through to containers. Install the NVIDIA container toolkit inside the
WSL distribution that runs Dokploy:

```bash
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey \
  | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg

curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list \
  | sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' \
  | sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

sudo apt-get update
sudo apt-get install -y nvidia-container-toolkit
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

That last command restarts the Docker daemon, which stops every running
container in the distribution, including Postgres and Redis. Redeploy afterwards.

Verify the runtime is registered before changing the compose file:

```bash
docker info | grep -i -A2 runtimes
```

The output must include an `nvidia` runtime alongside `runc`. If it does not,
enabling the compose block will reproduce the error:

```
could not select device driver "nvidia" with capabilities: [[gpu]]
```

Once `nvidia` appears, uncomment these three places in `docker-compose.yml`:

1. `NVIDIA_DRIVER_CAPABILITIES: all`
2. `NVIDIA_VISIBLE_DEVICES: all`
3. The `reservations.devices` block under `deploy.resources`

Then confirm the GPU reaches the container:

```bash
docker exec transcode nvidia-smi
```

Finally, switch the preset's encoder to NVENC, for example `hevc_nvenc` or
`h264_nvenc`.

### Rolling Back

Re-comment the same three blocks and restart the container. No host changes need
to be undone, since the toolkit installation is harmless while unused.

## Image Tag

The image defaults to `latest` because Unmanic does not publish version tags on
ghcr.io, only `latest`, `staging`, and development tags. Pin
`TRANSCODE_IMAGE` to a commit digest if you need reproducibility:

```bash
docker inspect --format '{{index .RepoDigests 0}}' ghcr.io/unmanic/unmanic:latest
```

## Security

Unmanic has no built-in authentication. Anyone who can reach port 8888 can queue
conversions that rewrite media files in place. Keep it on the LAN and restrict
the port with the host firewall.

## Resource Use

Memory is limited to 2G, which is sufficient for FFmpeg at typical resolutions.
No CPU limit is set on purpose, because throttling a conversion only makes it
take longer without reducing the total work done.

## Volumes to Back Up

Back up `stacks/files/transcode/config`, which holds presets and plugin
settings. The cache holds only in-progress files and does not need backing up.

## Alternatives

If you need more *arr integration or a larger plugin catalogue, Tdarr is the
other common choice. It imports libraries from Radarr and Sonarr, which requires
path translator configuration. Keeping the stack named `transcode` means
swapping the backing application later does not change the service name or
volume paths.