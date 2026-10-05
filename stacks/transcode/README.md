# Transcode

Library optimisation: converts and reorganises video files in place using
visual flows. Runs [FileFlows](https://fileflows.com), which points directly at
the media directories.

The stack is named generically so the service name stays independent of the
application it runs.

## Endpoint

| Item | Value |
|------|-------|
| Container port | 5000 |
| Published host port | 19200 → 5000 |
| Network | `dokploy-network` |
| Upstream address | `transcode:5000` |
| Reachable at | `http://<host-ip>:19200` |

This is an internal service and is not in the root `Caddyfile`. Host port 19200
is published so the UI is reachable from the LAN. Do not expose it publicly.

FileFlows defaults to 19200 on the host, which is why the mapping is
`19200:5000`. Change the left-hand side to move the host port.

## Why This Is Simpler Than the Alternatives

FileFlows is pointed straight at the media directories, so none of these apply:

- **No path translators.** Radarr and Sonarr are never queried, so their
  Windows paths never enter the picture.
- **No library import.** No API keys, no library type selection, no separate
  Movies and TV library definitions.
- **No node or worker to start.** The server image includes an internal agent,
  so a single container is a complete installation.
- **No plugin stack to write.** Flows are assembled from nodes in the UI and can
  be imported as JSON.

The trade-off is that initial setup is visual work in the browser rather than
configuration work in files.

## Volumes

| Host path | Container path | Purpose |
|-----------|----------------|---------|
| `../files/transcode/config` | `/app/Data` | settings, flows, library configuration |
| `../files/transcode/cache` | `/temp` | temporary conversion files |
| `/mnt/d/Entertainment/Movies` | `/library/Movies` | read/write, converts in place |
| `/mnt/d/Entertainment/TV` | `/library/TV` | read/write, converts in place |

All `../files/` paths resolve to `stacks/files/`, consistent with the rest of the
repository. They are bind mounts rather than named volumes so flows and settings
are directly readable and editable on the host.

Media mounts are read/write on purpose. FileFlows converts into `/temp` and then
replaces the original file.

`/mnt/d/Entertainment` is the WSL view of the Windows `D:` drive, which is what
the Docker daemon sees when Dokploy runs inside WSL2.

The cache holds full-size temporary files for every job. Keep
`stacks/files/transcode/cache` on local SSD rather than the media drive.

## First Run

1. Open `http://<host-ip>:19200`.
2. Under **Settings**, set the **Library Start Directory** to `/library/Movies`.
   This is the first path FileFlows offers to browse. Everything else is reached
   by moving up to `/library`.
3. Go to **Libraries** and add `/library/Movies` and `/library/TV`. These are
   container paths, not Windows host paths.
4. Confirm the scan finds files. An empty result means the media mount did not
   resolve; check with `docker exec transcode ls -ld /library/Movies /library/TV`.
5. Build a flow: **Flows → Library Optimisation**, then assemble a transcode flow
   from the video nodes. Start with a single encode node and no conditions.
6. Assign the flow to a library, then run it on **one** file and confirm the
   output plays in Jellyfin before enabling a schedule.

## Converting Only New Files

A FileFlows node watches for files added to a directory and can trigger a flow
when that happens. Point a watcher node at `/mnt/d/Entertainment/Import` to
convert incoming files before they are handed to Radarr or Sonarr.

`Import` is not mounted by default. Add it only if you want that behaviour:

```yaml
      - "/mnt/d/Entertainment/Import:/library/Import"
```

## Hardware Transcoding

**Current state: CPU only.** GPU passthrough is commented out because the NVIDIA
container toolkit is not installed in the WSL2 distribution running Dokploy.

CPU transcoding works out of the bundled FFmpeg and ImageMagick. In your flow,
set the video encoder to a software encoder such as `libx265`.

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

1. `NVIDIA_DRIVER_CAPABILITIES: compute,video,utility`
2. `NVIDIA_VISIBLE_DEVICES: all`
3. The `reservations.devices` block under `deploy.resources`

Then confirm the GPU reaches the container:

```bash
docker exec transcode nvidia-smi
```

Finally, change the encoder in your flow to NVENC, for example `hevc_nvenc` or
`h264_nvenc`.

Note that FileFlows uses `compute,video,utility` capabilities, not `all`. Using
`all` also works but grants more device access than needed.

### Rolling Back

Re-comment the same three blocks and restart the container. No host changes need
to be undone, since the toolkit installation is harmless while unused.

## Image Tag

`TRANSCODE_TAG` defaults to `stable`. FileFlows updates `latest` frequently, up
to daily, and it can carry experimental changes. Monthly stable releases are the
safer choice for unattended operation.

Other published tags:

| Tag | Contents |
|-----|----------|
| `stable` | monthly tested release, the default here |
| `latest` | most recent, frequently updated |
| `latest_modded` | bundles its own FFmpeg and ImageMagick |
| `modded_latest` | as above for stable releases |

Use a `modded` variant only if the container cannot reach the internet to fetch
DockerMods plugins.

## Jellyfin Integration

FileFlows ships a **Jellyfin Updater** flow node. It sends a request to Jellyfin
to refresh its library after a conversion, so Jellyfin picks up the new file
without waiting for a scheduled scan.

Install it under **Settings → Extensions → Plugins**, then configure it once
under the plugin's settings page so individual flows do not need it repeated:

| Setting | Value |
|---------|-------|
| Server | `http://host.docker.internal:8096` |
| Access Token | Jellyfin API key, from Dashboard → Advanced → API Keys |
| Mapping | see below |

**The mapping is required.** FileFlows sees the media at `/library/Movies`, but
Jellyfin has it registered as `D:\Entertainment\Movies`. Without a mapping,
Jellyfin cannot match the converted file to its library entry.

| FileFlows | Jellyfin |
|-----------|----------|
| `/library/Movies` | `D:\Entertainment\Movies` |
| `/library/TV` | `D:\Entertainment\TV` |

### Reaching Jellyfin

Jellyfin runs natively on the Windows host, so `127.0.0.1` inside the container
refers to the container itself, not Jellyfin. The stack maps
`host.docker.internal` to the Docker host gateway, which is why the plugin
settings use that hostname.

Verify it resolves before configuring the plugin:

```bash
docker exec transcode curl -s http://host.docker.internal:8096/System/Info/Public
```

A JSON response confirms the path works. If it fails, Jellyfin may be bound to
`127.0.0.1` only, in which case it must listen on the host LAN interface.

## Docker Siblings

Mounting `/var/run/docker.sock` enables FileFlows' Docker Siblings feature,
which lets it start and stop containerised jobs on demand. Leave it unmounted
unless you need it: a Docker socket mount grants broad control of the daemon,
and FileFlows does not require it for local transcoding.

```yaml
      - /var/run/docker.sock:/var/run/docker.sock:ro
```

If you enable it, `TempPathHost` must also be set so FileFlows can hand a host
path to the sibling container.

## Security

FileFlows has no built-in user accounts. Anyone who can reach port 19200 can
queue jobs that rewrite media files in place. Keep it on the LAN and restrict the
port with the host firewall.

An access token can be set under **Settings → Security**, but it governs
connections from external agents rather than web UI access, so it is not a
substitute for network restriction.

## Warnings

FileFlows rewrites media files in place. A badly built flow destroys source data
irreversibly. Back up `D:\Entertainment` before the first real run.

CPU transcoding is slow. An encode that the RTX 4050 would finish in minutes can
take most of an hour here.

## Resource Use

Memory is limited to 2G, which is sufficient for FFmpeg at typical resolutions.
Raise it for 4K remux work.

No CPU limit is set on purpose, because throttling a conversion only makes it
take longer without reducing the total work done.

## Volumes to Back Up

Back up `stacks/files/transcode/config`, which holds flows, settings, and library
configuration. The cache holds only in-progress files and does not need backing
up.