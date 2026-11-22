# SuggestArr

Suggests media based on watch history and sends requests to Seerr. Runs with host networking, so the UI is at `http://<host-ip>:5000`.

State is kept in the `suggestarr-config` named volume. Jellyfin and Seerr URLs and API keys are configured in the SuggestArr UI. Reach host services at `http://127.0.0.1:<port>`, for example Jellyfin at `http://127.0.0.1:8096` and Seerr at `http://127.0.0.1:5055`.
