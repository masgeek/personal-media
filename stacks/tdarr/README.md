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

GPU transcoding is enabled in the compose file through an NVIDIA device
reservation, alongside `NVIDIA_DRIVER_CAPABILITIES=all` and
`NVIDIA_VISIBLE_DEVICES=all`.

Verify the GPU is visible to the container before configuring any plugins:

```bash
docker exec tdarr nvidia-smi
```

If that command is missing or reports no devices, the NVIDIA container runtime
is not registered with this Docker daemon. Under WSL2 this normally means the
Windows NVIDIA driver is not installed, or the NVIDIA container toolkit is not
installed in the WSL distribution running Dokploy.

Once the GPU is visible, set the transcode actions in Tdarr to use NVENC rather
than the CPU encoder, and create a GPU worker on the Tdarr tab.

The `/dev/dri` device mapping used for Intel and AMD is intentionally not set.
It would fail to start on a machine without that device path.

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