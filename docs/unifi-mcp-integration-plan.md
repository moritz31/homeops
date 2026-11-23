# UniFi MCP deployment

The draft in `kubernetes/apps/default/unifi-mcp` uses [sirkirby's Network MCP server](https://github.com/sirkirby/unifi-mcp/blob/main/apps/network/README.md). The originally requested [onedr0p deployment](https://github.com/onedr0p/home-ops/tree/main/kubernetes/apps/default/unifi-mcp) could not be retrieved, so this is an independently adapted implementation. It is deliberately not registered in `kubernetes/apps/default/kustomization.yaml` until the secret is populated and encrypted.

The HelmRelease connects to the existing standalone UniFi Network service at `unifi.default.svc.cluster.local:8443`, with site `default`. It provides a ClusterIP service on port 3000 and Streamable HTTP at `/mcp`, with a bearer token and read-only mode. The service is HTTP inside the cluster; use port-forward for local access, or add a properly authenticated TLS proxy and restrict backend access before remote use. No public or internal Gateway route is created.

## Prepare the controller and secret

1. Use your dedicated local read-only UniFi Network account. Confirm that it can sign in to the standalone controller and has access to the intended site. Change `UNIFI_NETWORK_SITE` if its internal site identifier is not `default`. The server supports username and password session authentication; an API key is not required for this path. See the [Network authentication documentation](https://github.com/sirkirby/unifi-mcp/blob/main/apps/network/README.md).
2. Obtain and independently verify the SHA-256 fingerprint of the certificate presented by the controller on port 8443. For example, from a trusted machine with network access to the controller, replace `CONTROLLER_ADDRESS` with its direct address and run:

   ```sh
   openssl s_client -connect CONTROLLER_ADDRESS:8443 -servername unifi.default.svc.cluster.local </dev/null 2>/dev/null | openssl x509 -noout -fingerprint -sha256
   ```

   Ensure this is the same certificate presented by the Kubernetes service. Certificate pinning overrides normal certificate verification; update the pin when the certificate changes. See [upstream TLS documentation](https://github.com/sirkirby/unifi-mcp/blob/main/docs/tls-verification.md).
3. In `app/unifi-mcp-secret.sops.yaml`, replace the username, password, fingerprint, and bearer-token placeholders. Generate a token using `openssl rand -hex 32`. The template is plaintext until encrypted; never commit or apply real plaintext values. From the repository root run:

   ```sh
   sops --encrypt --in-place kubernetes/apps/default/unifi-mcp/app/unifi-mcp-secret.sops.yaml
   ```

   The repository's SOPS rules encrypt `stringData` with the configured age recipient. Flux injects its `sops-age` decryption settings into child Kustomizations.
4. The image currently uses upstream's documented `latest` tag because a versioned GHCR tag and digest could not be verified here. Before activation, resolve and pin a release image and digest that supports the documented session credentials, fingerprint, bearer-token and read-only settings. Verify the image's architecture matches the nodes.
5. Add `- ./unifi-mcp/ks.yaml` to `kubernetes/apps/default/kustomization.yaml` and commit the encrypted secret with the manifests. Flux will wait for `cluster-apps-unifi`, create the secret and HelmRelease, and report the HelmRelease health.

## Verify and connect

Check the Flux Kustomization, HelmRelease, pod and logs. TCP probes only confirm that the HTTP listener is running; they do not prove controller authentication. After rollout, use:

```sh
kubectl -n default port-forward svc/unifi-mcp 3000:3000
```

Configure an MCP client for Streamable HTTP at `http://127.0.0.1:3000/mcp` with `Authorization: Bearer <token>`. Initialize the session, list tools, and perform a harmless inventory read. Confirm unauthenticated requests are rejected and writes are denied. If inventory fails, inspect logs without sharing credentials, verify the certificate pin, account credentials, site access and standalone controller compatibility. Keep read-only mode enabled and restart the pod after secret changes.

For in-cluster clients, the service URL is `http://unifi-mcp.default.svc.cluster.local:3000/mcp`; HTTP sends the token unencrypted on that path. Use TLS and access controls if the cluster network is not trusted. Secret changes supplied through environment variables require restarting the deployment; TCP readiness does not test the UniFi API.
