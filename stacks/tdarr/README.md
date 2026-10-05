# Tdarr

Library analytics and transcode automation for Radarr and Sonarr libraries.
Applies plugin stacks to normalise codecs and containers, most often to convert
H.264 to H.265 and cut file size by roughly 40–50%.

Configuration follows the official
[Run and Compose guide](https://docs.tdarr.io/docs/installation/docker/run-compose/).

## Endpoint

| Item | Value |
|------|-------|
| Web UI port | 8265 |
| Server port | 8266, nodes connect outbound to this |
| Network | `host` |
| Reachable at | `http://<host-ip>:8265` |

Tdarr is an internal service and is not in the root `Caddyfile`. It is reached
on the host port directly. Do not expose it publicly.

## Why Host Networking

Tdarr imports libraries from Radarr and Sonarr, which run directly on the
Windows host. Host networking lets it reach them at `http://127.0.0.1:7878` and
`http://127.0.0.1:8989`, which a bridge network cannot do.

Because host networking is used, there is no `ports` block. Ports 8265 and 8266
are bound directly on the host.

## Volumes

| Host path | Container path | Purpose |
|-----------|----------------|---------|
| `../files/tdarr/server` | `/app/server` | server database, samples, plugins |
| `../files/tdarr/configs` | `/app/configs` | `Tdarr_Server_Config.json` and `Tdarr_Node_Config.json` |
| `../files/tdarr/logs` | `/app/logs` | application logs |
| `../files/transcode/tdarr` | `/temp` | working files during transcode |
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

The transcode cache holds full-size temporary files for every job. Keep
`stacks/files/transcode/tdarr` on local SSD rather than the media drive.

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

This is the step most likely to cause confusion. Tdarr imports library paths
verbatim from Radarr and Sonarr, which report Windows paths like
`D:\Entertainment\Movies\...`. Those paths do not exist inside the container, so
every file fails to resolve and the queue stays empty with no error.

`pathTranslators` has no environment variable equivalent and must be set in
`stacks/files/tdarr/configs/Tdarr_Node_Config.json`:

```json
"pathTranslators": [
  { "server": "D:\\Entertainment\\Movies", "node": "/movies" },
  { "server": "D:\\Entertainment\\TV", "node": "/tv" }
]
```

Backslashes in JSON must be escaped as `\\`. Restart the container afterwards,
or Tdarr will overwrite the file on start.

## Hardware Transcoding

**Current state: CPU only.** GPU passthrough is commented out in the compose
file because the NVIDIA container toolkit is not installed in the WSL2
distribution running Dokploy. See "Enabling NVIDIA Later" below.

CPU transcoding works out of the box with the bundled FFmpeg 7 and HandBrake.
Set the transcode actions in Tdarr to use a software encoder such as `libx265`,
and start with a single CPU worker.

Your RTX 4050's NVENC encoder is substantially faster than software encoding,
so enabling the GPU is worth doing when convenient.

## Enabling NVIDIA Later

The GPU is already visible to WSL2 itself. Confirm that first:

```bash
nvidia-smi -L
```

If that lists the card, the remaining problem is only that Docker cannot pass
the device through to containers. Install the NVIDIA container toolkit inside
the WSL distribution that runs Dokploy:

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
the toolkit is not wired up and enabling the compose block will reproduce the
error:

```
could not select device driver "nvidia" with capabilities: [[gpu]]
```

Once `nvidia` appears, uncomment these three places in `docker-compose.yml`:

1. `NVIDIA_DRIVER_CAPABILITIES: all`
2. `NVIDIA_VISIBLE_DEVICES: all`
3. The `reservations.devices` block under `deploy.resources`

Then confirm the GPU reaches the container:

```bash
docker exec tdarr nvidia-smi
```

Finally, set the transcode actions to an NVENC encoder such as `hevc_nvenc` or
`h264_nvenc` and create a GPU worker on the Tdarr tab.

### Rolling Back

Re-comment the same three blocks and restart the container. No host changes need
to be undone, since the toolkit installation is harmless while unused.

## First Run

1. Open `http://<host-ip>:8265`.
2. Confirm a Node appears on the Tdarr tab. If not, `internalNode` is not set.
3. Add libraries from Radarr and Sonarr using their API keys. The URLs
   `http://127.0.0.1:7878` and `http://127.0.0.1:8989` are correct because host
   networking shares the host loopback interface.
4. Set path translators as described above.
5. Confirm the binary tests pass. The image ships FFmpeg and HandBrake. If a test
   fails, set `ffmpegPath`, `handbrakePath`, or `mkvpropeditPath` in the node
   config.
6. Start a single CPU worker against a small library before enabling a schedule.

## Security

`auth` is set to `false`, so the Tdarr UI has no login. Anyone who can reach port
8265 can start transcode jobs that rewrite your media files in place. Keep it on
the LAN and restrict the port with the host firewall, or set `auth=true` and
configure a secret key in the server config.

## Resource Use

Transcoding saturates the CPU and runs for a long time. Memory is limited to 2G,
which is sufficient for FFmpeg at typical resolutions. Raise it for 4K remux
work.

No CPU limit is set on purpose. Throttling a transcode worker makes jobs take
longer without reducing the total work done.

## Volumes to Back Up

Back up `stacks/files/tdarr/`, particularly `server` and `configs`. Losing
`configs` loses all plugin stacks and path translators, which is tedious to
rebuild. The transcode cache holds only in-progress files and does not need
backing up.