# Public ingress to Gateway API migration

**Repository cleanup:** The current manifests remove the Ingress resources and the Immich bridge. The steps below describe the prior staged migration and rollback procedure; restoring nginx routing now requires reverting that cleanup in Git. Keep the nginx controllers until their TCP and UDP forwards have been migrated and verified.

The repository stages the public HTTPRoutes alongside the existing nginx Ingresses. Deploy these changes before changing the router. Until the router is switched, nginx remains the public entry point; Immich uses its existing nginx-to-Envoy bridge. Do not remove the Ingresses or the bridge before the cutover. No router settings are changed by this repository.

## Prepare and test

1. Back up Authelia, Paperless and UniFi data before deploying their app-template chart upgrades. The upgrades retain their PVC names and preserve their existing workload selectors; UniFi keeps the same LoadBalancer Service and device ports. check their HelmRelease status and application logs after reconciliation. Commit and push the changes and wait for Flux to reconcile. Check the Envoy Gateway HelmRelease and the public Gateway:

   ```sh
   flux get kustomizations -A
   flux get helmreleases -A
   kubectl -n networking get gateway public -o yaml
   kubectl -n networking get certificate kirstein-me-public-gateway
   kubectl get httproutes -A
   kubectl -n networking get svc envoy-public ingress-external-nginx-controller -o wide
   ```

   Authelia, Paperless and UniFi now have Helm-managed routes named `authelia-gateway`, `paperless-gateway` and `unifi-gateway`, plus matching `-redirect` routes. If the older standalone routes were already deployed, Flux prunes them; check that the replacements are accepted, since reconciliation order can cause a brief gap. Confirm the Gateway is programmed, the certificate is ready, the Envoy service has a reachable load balancer IP, and each public route has `Accepted=True` and `ResolvedRefs=True` for the intended listener. Use `kubectl -n <namespace> describe httproute <name>` for details. Verify both the HTTPS route and the HTTP redirect route. The exact nginx service name can be found with `kubectl -n networking get svc` if it differs.

2. From a machine able to reach the Envoy load balancer address, test each hostname directly, preserving TLS SNI and Host headers. Substitute the actual IPv4 address from `envoy-public`:

   ```sh
   GATEWAY_IP=<envoy-public-load-balancer-ip>
   for host in auth paperless unifi grafana jellyfin jellyseerr photos flux-webhook; do
     curl -sS -o /dev/null -D - --resolve "$host.kirstein.me:80:$GATEWAY_IP" "http://$host.kirstein.me/"
     curl -sS -o /dev/null -D - --resolve "$host.kirstein.me:443:$GATEWAY_IP" "https://$host.kirstein.me/"
   done
   ```

   HTTP should redirect to HTTPS. Verify each HTTPS application, login, uploads and streaming in a browser with equivalent local DNS overrides. Grafana is configured with zero replicas, so it cannot pass an application test until scaled up. The webhook only matches `/hook/`: a request to `/` is expected to return 404. Test its actual receiver path with a GitHub redelivery after cutover; obtain it from `kubectl -n flux-system get receiver github-receiver -o yaml` without sharing the token or path. Confirm Paperless uploads up to the desired size; nginx's old 10 MB limit is not reproduced by the HTTPRoute.

   UniFi uses an Envoy `Backend` with TLS certificate verification disabled because its backend normally presents a self-signed certificate. This encrypts the connection but does not authenticate the backend; configure a trusted UniFi certificate and a `BackendTLSPolicy` if verification is required. Check the UniFi UI specifically before cutover. See [Envoy's backend TLS documentation](https://gateway.envoyproxy.io/v1.10/tasks/security/backend-skip-tls-verification/).

3. Record the router's current TCP 80 and 443 forwarding destinations and the nginx service IP. Ensure the Envoy load balancer IP is stable, reserving it if necessary. Public DNS is already targeted at `home.kirstein.me` via ExternalDNS; inspect the actual public DNS records and confirm that name still resolves to your public WAN address. DNS alone does not switch the router's internal forwarding destination. Check split-horizon/internal DNS separately.

## Switch and verify

1. Change the router's **TCP 80 and TCP 443** port forwards from the nginx load balancer IP to the `envoy-public` load balancer IP. Do not change UniFi device-inform TCP 8080 or STUN UDP 3478 forwards as part of this step: nginx currently proxies both to `default/unifi`, and Envoy's public Gateway only listens on 80 and 443.
2. From outside the home network, verify HTTPS certificates, HTTP redirects, each application and the webhook. Exercise sign-in, Paperless and Immich uploads, and Jellyfin streaming. Verify UniFi devices still connect and STUN works. Check Gateway/route conditions and Envoy logs if any route fails.
3. If anything critical fails, restore the original router TCP 80/443 destinations. The staged nginx Ingresses and Immich bridge remain available for rollback. Investigate and repeat the direct-IP tests before trying again. If you have already removed the old resources, revert that cleanup in Git and wait for Flux to restore them first.

## Cleanup after a successful cutover

After verification and an appropriate rollback window, commit a follow-up cleanup:

- Disable/remove the `ingress` values in the Authelia, Paperless, UniFi, Jellyfin and Jellyseerr HelmReleases. Paperless already has `PAPERLESS_URL` set to `https://paperless.kirstein.me`.
- Remove `spec.ingress` from the Grafana resource. Remove the Flux webhook `ingress.yaml` from its kustomization and delete the file.
- Remove Immich's `ingress-bridge.yaml` from its kustomization and delete the file; Flux pruning will remove its bridge Service, Certificate and Ingress.
- Remove UniFi's dependency on `cluster-apps-ingress-external-nginx` when no longer needed. Keep nginx running until the TCP 8080 and UDP 3478 forwards have been moved to another working destination and verified. Only then remove its Flux configuration if no other consumers remain.
- Review ExternalDNS ingress sources only after checking for any remaining Ingress consumers and DNS records. Reconcile Flux and verify routes, DNS and application traffic again.

The wildcard certificate on the public Gateway covers the listed `*.kirstein.me` hosts. Do not delete other certificates or DNS records without checking their consumers.
