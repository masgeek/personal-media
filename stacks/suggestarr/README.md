# SuggestArr

Suggests media based on watch history and sends requests to Seerr. UI at `http://<host-ip>:5000`.

State is kept in the `suggestarr-config` named volume. Jellyfin and Seerr URLs and API keys are configured in the SuggestArr UI. From the container, reach host services at `http://host.docker.internal:<port>`.
