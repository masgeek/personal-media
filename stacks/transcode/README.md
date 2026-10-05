# Transcode

Library optimisation: converts video files in place to a single uniform format.
Runs [Tdarr](https://tdarr.io), which imports libraries directly from Radarr and
Sonarr and applies plugin stacks to normalise codecs and containers.

The stack is named generically so the backing application can be swapped later
without renaming the service or the volume paths. See **Alternatives** at the
bottom.

Configuration follows the official
[Run and Compose guide](https://docs.tdarr.io/docs/installation/docker/run-compose/).

## Endpoint

| Item | Value |
|------|-------|
| Web UI port | 8265 |
| Server port | 8266, nodes connect outbound to this |
| Network | `host` |
| Reachable at | `http://<host-ip>:8265` |

This is an internal service and is not in the root `Caddyfile`. It is reached on
the host port directly. Do not expose it publicly.

Because host networking is used, there is no `ports` block. Ports 8265 and 8266
are bound directly on the host.

## Why Host Networking

Tdarr imports libraries from Radarr and Sonarr, which run directly on the
Windows host. Host networking lets it reach them at `http://127.0.0.1:7878` and
`http://127.0.0.1:8989`, which a bridge network cannot do.

The tradeoff is that Tdarr is not attached to `dokploy-network`, so Dokploy
cannot route to it by container name.

## Volumes

| Host path | Container path | Purpose |
|-----------|----------------|---------|
| `../files/transcode/server` | `/app/server` | server database, samples, plugins |
| `../files/transcode/configs` | `/app/configs` | `Tdarr_Server_Config.json` and `Tdarr_Node_Config.json` |
| `../files/transcode/logs` | `/app/logs` | application logs |
| `../files/transcode/cache` | `/temp` | working files during transcode |
| `/mnt/d/Entertainment/Movies` | `/movies` | read/write, transcodes in place |
| `/mnt/d/Entertainment/TV` | `/tv` | read/write, transcodes in place |

All `../files/` paths resolve to `stacks/files/`, consistent with the rest of the
repository. These are bind mounts rather than named volumes because
`pathTranslators` can only be set by editing `Tdarr_Node_Config.json`, which is
far easier when the file is reachable on the host.

Media mounts follow the same pattern as the `bazarr` stack: host media paths are
mounted at simplified container paths rather than identical ones. Media mounts
are read/write on purpose, because Tdarr rewrites files in place after
transcoding into the cache directory and moving the result back.

`/mnt/d/Entertainment` is the WSL view of the Windows `D:` drive, which is what
the Docker daemon sees when Dokploy runs inside WSL2.

The cache holds full-size temporary files for every job. Keep
`stacks/files/transcode/cache` on local SSD rather than the media drive.

## Required Configuration

These environment variables are set in the compose file and should not be
removed:

| Variable | Why it matters |
|----------|----------------|
| `internalNode=true` | runs a Node inside the Server container. Without it the server starts but never transcodes anything |
| `inContainer=true` | tells Tdarr it is containerised |
| `serverIP=0.0.0.0` | required for nodes to reach the server |
| `serverPort=8266` | node connection port |
| `webUIPort=8265` | web UI port |
| `ffmpegVersion=7` | bundled FFmpeg major version |

Optional: `TDARR_VERSION` (defaults to `2.94.02`), `TDARR_PUID`, `TDARR_PGID`,
`TDARR_NODE_NAME`, and the root `TZ`.

`openBrowser` is set to `false` because there is no browser in the container.

## Path Translators

This is the step most likely to cause confusion, and it is mandatory. Tdarr
imports library paths verbatim from Radarr and Sonarr, which report Windows
paths like `D:\Entertainment\Movies\...`. Those paths do not exist inside the
container, so every file fails to resolve and the queue stays empty with no
error.

`pathTranslators` has no environment variable equivalent and must be set in
`stacks/files/transcode/configs/Tdarr_Node_Config.json`:

```json
"pathTranslators": [
  { "server": "D:\\Entertainment\\Movies", "node": "/movies" },
  { "server": "D:\\Entertainment\\TV", "node": "/tv" }
]
```

`path-translators.example.json` in this directory holds the same block as a
template, with the `server` values pre-filled for this layout.

Backslashes in JSON must be escaped as `\\`. Restart the container afterwards,
or Tdarr will overwrite the file on start.

## Walkthrough: Adding Libraries and Transcoding

Follow this order. Setting path translators first is not optional, because
libraries imported before that resolve to zero files with no visible error.

### 1. Preflight

```bash
docker exec transcode ls -ld /movies /tv
```

Both must exist. If empty, the media mounts did not resolve.

Then open `http://<host-ip>:8265` and check the **Tdarr** tab. A node named
`internal-node` must be listed. No node means `internalNode` is not set, and
nothing will ever transcode.

### 2. Set Path Translators

```bash
docker compose -f stacks/transcode/docker-compose.yml stop transcode
```

Edit `stacks/files/transcode/configs/Tdarr_Node_Config.json` using the block
above, then:

```bash
docker compose -f stacks/transcode/docker-compose.yml start transcode
```

The container must be stopped while editing. Tdarr rewrites this file on start
and overwrites changes made while running.

### 3. Add Libraries

Get the API keys from Radarr and Sonarr under Settings → General.

In Tdarr, go to **Libraries → New Library**:

| Field | Movies library | TV library |
|-------|----------------|------------|
| Library type | Radarr | Sonarr |
| URL | `http://127.0.0.1:7878` | `http://127.0.0.1:8989` |
| API key | Radarr's API key | Sonarr's API key |
| Path | `/movies` | `/tv` |

`127.0.0.1` is correct because host networking shares the host loopback
interface. A bridge network would not reach the Windows-hosted services there.

**Verify before continuing.** Open the library and confirm it lists real files.
An empty library means the path translators did not apply, so return to step 2.

### 4. Build a Plugin Stack

Go to **Plugin Stacks → New**. A reasonable first stack for CPU transcoding:

1. **Condition** — Video Codec is not `hevc`
2. **Action** — Transcode, encoder `libx265`, preset `medium`

Add a remux condition later if container normalisation is also wanted. Start
with a single condition and action, since a complex stack is difficult to debug
when it produces unexpected output.

### 5. Start a Worker

Go to the **Tdarr tab → New Worker**:

| Setting | Value |
|---------|-------|
| Worker type | Transcode CPU |
| Workers | `1` |
| Start paused | No |

Use one worker. Software transcoding saturates the host, so additional workers
only divide the same cores without finishing sooner.

### 6. Test on a Single File

Assign the plugin stack to the library, then run it against **one** item and
watch the log. Confirm the resulting file plays in Jellyfin before touching the
whole library.

### 7. Schedule

Once a manual run is verified, set the library schedule and start a fresh
library with the stack assigned.

## Licensing

Tdarr is proprietary, with a free tier that is sufficient for single-machine
transcoding. The free tier includes the server, up to 5 nodes, plugin stacks and
flows, CPU and GPU workers, health checks, and the community plugin catalogue.

The paid tier ($4.99/month or $49.99/year) only adds Tdarr Relay, duplicate
file finder, size explorer, extra statistics, node prioritisation,
library-to-node assignment, node tags, unmapped nodes, and Discord
notifications. None of those are needed to convert a library on one machine.

The trade-off is that Tdarr is source-available rather than open source, so the
core function depends on the vendor and the free tier's contents are not
guaranteed to stay unchanged.

## Hardware Transcoding

**Current state: CPU only.** GPU passthrough is commented out because the NVIDIA
container toolkit is not installed in the WSL2 distribution running Dokploy.

CPU transcoding works out of the bundled FFmpeg 7 and HandBrake. Set transcode
actions to a software encoder such as `libx265`, and start with a single CPU
worker.

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

Finally, set the transcode actions to an NVENC encoder such as `hevc_nvenc` or
`h264_nvenc`, and create a GPU worker on the Tdarr tab.

### Rolling Back

Re-comment the same three blocks and restart the container. No host changes need
to be undone, since the toolkit installation is harmless while unused.

## Security

`auth` is set to `false`, so the Tdarr UI has no login. Anyone who can reach port
8265 can start transcode jobs that rewrite your media files in place. Keep it on
the LAN and restrict the port with the host firewall, or set `auth=true` and
configure a secret key in the server config.

## Warnings

Tdarr transcodes into the cache directory and then moves the result back over
the original file. A wrong plugin stack destroys source data irreversibly. Back
up `D:\Entertainment` before the first real run.

CPU transcoding is slow. An encode that the RTX 4050 would finish in minutes can
take most of an hour here.

## Resource Use

Memory is limited to 2G, which is sufficient for FFmpeg at typical resolutions.
Raise it for 4K remux work.

No CPU limit is set on purpose, because throttling a transcode worker makes jobs
take longer without reducing the total work done.

## Volumes to Back Up

Back up `stacks/files/transcode/`, particularly `server` and `configs`. Losing
`configs` loses all plugin stacks and path translators, which is tedious to
rebuild. The cache holds only in-progress files and does not need backing up.
