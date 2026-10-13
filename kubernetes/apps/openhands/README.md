# OpenHands

Deployed using the `bjw-s-labs/app-template` Helm chart, matching existing Flux HelmReleases in this repository. The official [Agent Canvas chart](https://github.com/OpenHands/OpenHands/tree/main/helm/agent-canvas) deploys the full local stack and does not expose a container command override, so app-template is used for frontend-only operation.

Agent Canvas runs in frontend-only mode at https://openhands.int.kirstein.me/canvas/ through the internal ingress. The image entrypoint is overridden so no local Agent Server, automation service, editor, or Codex process starts.

Add a remote OpenHands Agent Server backend in the UI, enter its API key, and select Codex as the agent. The remote server must provide Codex/ACP support. Backend addresses must be reachable from your browser; HTTPS backends must allow the Canvas origin (`https://openhands.int.kirstein.me`) for CORS and support WebSockets. Scheduled automations require an automation service on the remote backend.

Canvas backend connections are stored in browser local storage. Workspaces, credentials, and conversation persistence belong to the remote backend, so this deployment needs no PVC, Docker socket, or LLM secrets.

Initial resources: one replica, 10m CPU and 64Mi memory requested, 256Mi memory limit. These are starting values for the static server, not measured production requirements.

Upstream references: [Agent Canvas](https://github.com/OpenHands/OpenHands), [frontend-only routing](https://github.com/OpenHands/OpenHands/blob/main/scripts/dev-with-automation.mjs), and [container layout](https://github.com/OpenHands/OpenHands/blob/main/docker/Dockerfile).
