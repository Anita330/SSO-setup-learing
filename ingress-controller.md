# Kubernetes Ingress Controller

This guide explains what Ingress is, the main controller choices, how HTTP/HTTPS traffic reaches a Service, and how to deploy and update the configuration.

> **Current direction:** Kubernetes keeps the `networking.k8s.io/v1` Ingress API stable, but the API is frozen. Kubernetes recommends Gateway API for new deployments. The community **Ingress-NGINX** controller was retired on March 24, 2026; existing installations continue to run, but receive no security or bug-fix releases. Do not select it for a new production cluster. “NGINX Ingress Controller” from F5 is a separate project. See the [Kubernetes Ingress documentation](https://kubernetes.io/docs/concepts/services-networking/ingress/), [controller list](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/), and [retirement notice](https://kubernetes.io/blog/2025/11/11/ingress-nginx-retirement/).

## 1. Ingress resource vs. Ingress controller

- **Ingress** is a Kubernetes API resource containing HTTP/HTTPS routing rules: hostnames, URL paths, TLS references, and backend Services.
- **Ingress controller** is the running implementation that watches Ingress resources and makes those rules work. It may provision a cloud load balancer, configure a proxy, or integrate with a network appliance.
- **IngressClass** identifies which controller should process an Ingress. Set `spec.ingressClassName` explicitly when possible.

Creating an Ingress resource alone does not expose an application. A compatible controller must be installed, reachable from clients, and configured to watch that resource/class.

Ingress handles HTTP and HTTPS. For other protocols or arbitrary ports, use an appropriate `Service` (`LoadBalancer` or `NodePort`) or a controller-specific feature.

## 2. Common controller and exposure choices

These are different dimensions: choose a controller implementation, then choose how its data-plane endpoint is exposed.

### Controller implementations

| Choice | Typical fit | Notes |
| --- | --- | --- |
| Cloud-provider controller (for example AWS Load Balancer Controller, GKE Ingress, or an equivalent for your cloud) | Clusters where the cloud load balancer should be provisioned and managed from Kubernetes | Integrates with provider networking and often supports provider-specific annotations or custom resources. Check provider and cluster prerequisites. |
| Maintained third-party proxy controller (for example F5 NGINX Ingress Controller, Traefik, HAProxy, Kong, or Envoy-based controllers) | On-premises, multi-cloud, or teams needing a particular proxy/API-gateway feature set | Installation, annotation behavior, licensing, and supported APIs vary. Validate the vendor's current support and security policy. |
| Gateway API implementation | New platforms that need a richer, role-oriented routing API | Preferred Kubernetes direction for new work. Install a Gateway API implementation; Gateway API CRDs by themselves do not route traffic. |

The community project named **Ingress-NGINX** is retired. It is not the same project as F5's **NGINX Ingress Controller**. Review the official [controller options](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/) and the selected implementation's documentation before choosing.

### How the endpoint is exposed

- **`Service` type `LoadBalancer`:** A cloud integration or load-balancer implementation gives the controller a stable external address. This is common in managed cloud clusters.
- **NodePort:** Opens a port on each node. A separate external load balancer, firewall, or router must send traffic to those nodes.
- **Host network / host ports:** The controller binds directly to node networking. This is environment-specific and requires careful port, scheduling, and security planning.
- **External appliance or edge proxy:** A network device forwards traffic to the cluster/controller endpoint.

These choices determine how traffic reaches the controller; they do not change the Ingress routing rules themselves.

## 3. Request flow

1. A DNS record points `app.example.com` to the controller's external address (or to an upstream load balancer that reaches it).
2. The client connects over HTTP (port 80) or HTTPS (port 443).
3. The cloud load balancer, node-level entry point, or edge proxy forwards the request to the controller's Service and controller Pods.
4. The controller watches the Kubernetes API, finds the Ingress whose class, host, and path match, and applies the corresponding routing configuration.
5. For HTTPS, the controller uses the referenced TLS Secret to present the certificate. TLS may instead terminate at an upstream load balancer if that is the chosen design.
6. The controller forwards the request to the matching Kubernetes Service and port.
7. The Service sends it to a ready backend Pod selected by its labels/endpoints. The response returns through the controller to the client.

```text
Client -> DNS -> external LB / edge -> controller Service -> controller Pod
                                                        -> Ingress host/path rule
                                                        -> backend Service -> ready Pod
```

The Ingress points to a **Service**, not directly to a Pod. This lets Kubernetes update the backend endpoints as Pods are replaced or scaled.

## 4. Example Ingress configuration

The following manifest assumes a controller is already installed and has an IngressClass named `traefik`. Change the class and backend names to match your cluster. The `web` Service must exist in the same namespace and expose port 8080.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web
  namespace: default
spec:
  ingressClassName: traefik
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web
                port:
                  number: 8080
```

For HTTPS, create a TLS Secret in the same namespace as the Ingress, then add:

```yaml
  tls:
    - hosts:
        - app.example.com
      secretName: app-example-com-tls
```

Create a TLS Secret from a certificate and private key (keep the key out of source control):

```bash
kubectl -n default create secret tls app-example-com-tls \
  --cert=fullchain.pem --key=privkey.pem
```

`pathType: Prefix` matches a path prefix according to Kubernetes semantics. `Exact` matches the full path; `ImplementationSpecific` delegates matching behavior to the controller. Controller annotations and custom resources are implementation-specific and may not transfer to another controller.

## 5. Deployment workflow

Controller installation is platform-specific, so there is no single safe command that works for every cluster. Use this workflow and the current installation guide for the selected implementation.

### Before installation

1. Identify the Kubernetes distribution/version, cloud or on-premises environment, network plugin, DNS provider, and who owns external IP/load-balancer provisioning.
2. Choose a supported controller and its supported installation method (for example, the vendor's Helm chart or managed add-on). For new deployments, evaluate Gateway API implementations as well.
3. Decide whether the controller is internal or internet-facing, how it gets an external address, whether TLS terminates at the controller or upstream, and which namespaces/IngressClasses it watches.
4. Review required RBAC, admission webhooks, chart values, resource requests, replica count, tolerations/affinity, security settings, and upgrade policy. Pin a reviewed chart and image version; avoid floating `latest` tags.
5. Plan DNS, certificate issuance/renewal, firewall rules, health checks, observability, and rollback before exposing production traffic.

### Install the controller

For Helm-based controllers, the general pattern is:

```bash
helm repo add <repo-name> <official-chart-repository>
helm repo update
helm show values <repo-name>/<chart-name> --version <version>
helm upgrade --install <release-name> <repo-name>/<chart-name> \
  --namespace <controller-namespace> --create-namespace \
  --version <version> -f values.yaml
```

Replace placeholders with values from the chosen controller's official install guide. The chart and configuration keys are not interchangeable across projects. Store reviewed values in version control, excluding credentials and private keys.

Check that the controller Pods, Service, and class are ready:

```bash
kubectl get pods,service -n <controller-namespace>
kubectl get ingressclass
kubectl describe ingressclass <class-name>
```

Wait for an external IP/hostname if using a load balancer. If it remains pending, inspect cloud-controller/load-balancer events and the provider integration configuration.

### Deploy the application and route

Deploy the app's Deployment and ClusterIP Service first, then apply the Ingress manifest:

```bash
kubectl apply -f app-deployment.yaml
kubectl apply -f app-service.yaml
kubectl apply -f app-ingress.yaml
```

Verify resources and DNS:

```bash
kubectl get deploy,pods,svc,ingress -n default
kubectl describe ingress web -n default
kubectl get endpointslice -n default
```

Once DNS resolves to the entry point, test routing with `curl -i https://app.example.com/`. During DNS setup, a temporary test can target the controller address while supplying the host header, for example `curl -i -H 'Host: app.example.com' http://<external-address>/`.

## 6. Updating and operating ingress

### Change a route

Edit the manifest in version control and apply it:

```bash
kubectl diff -f app-ingress.yaml
kubectl apply -f app-ingress.yaml
kubectl describe ingress web -n default
```

The controller watches the API and reconciles changes. Check controller logs/events and test the route. Keep changes small so failures are easy to diagnose and roll back with the previous manifest revision.

### Change a TLS certificate

Update the TLS Secret using your certificate automation system or a controlled secret update. Ensure the Secret is in the same namespace and uses the name referenced by the Ingress. Verify certificate subject, chain, expiration, and served certificate after reconciliation. Do not commit private keys.

### Upgrade the controller

1. Read the selected controller's release notes and compatibility matrix for Kubernetes, chart, CRDs, and configuration changes.
2. Review breaking changes and deprecated annotations; back up Helm values and current manifests. Test in a non-production cluster.
3. Pin the target chart/image version and apply the upgrade using the vendor's procedure. For a Helm installation, this commonly uses `helm upgrade --install` with the new `--version` and reviewed values.
4. Watch rollout health, controller events/logs, external health checks, TLS, and representative routes. Keep the previous chart version and values available for rollback.
5. If using Gateway API CRDs or controller-specific CRDs, follow the vendor's CRD upgrade and rollback ordering; CRDs are cluster-scoped and require particular care.

### Useful checks

```bash
kubectl get ingress -A
kubectl get ingressclass
kubectl describe ingress <name> -n <namespace>
kubectl get svc,pods -n <controller-namespace>
kubectl logs -n <controller-namespace> deploy/<controller-deployment> --since=15m
kubectl get events -A --sort-by=.lastTimestamp
```

Common causes of failure:

- No controller is installed, the controller is unhealthy, or it does not watch the Ingress's class/namespace.
- `ingressClassName` is missing or names a class that does not exist.
- The Ingress backend Service name/port is wrong, or the Service has no ready EndpointSlices.
- DNS points to the wrong address, the external load balancer/firewall cannot reach the controller, or the load balancer address is still pending.
- Host/path does not match the request; path behavior or annotations differ between controller implementations.
- TLS Secret is missing, in the wrong namespace, malformed, expired, or does not cover the hostname.
- NetworkPolicy, cloud security groups, or firewall rules block traffic between the controller and backend Pods.

## 7. Production checklist

- [ ] Controller is supported, maintained, version-pinned, and installed from its official source.
- [ ] IngressClass is explicit and each Ingress is handled by the intended controller.
- [ ] External exposure, DNS, firewall/security groups, and health checks are configured and owned.
- [ ] TLS certificate issuance, renewal, storage, and expiry monitoring are in place.
- [ ] Controller has appropriate replicas, resource sizing, disruption handling, and observability.
- [ ] Routes are tested for expected host/path behavior and unauthorized/default-host requests.
- [ ] Controller annotations/custom resources and upgrade/rollback steps are documented.
- [ ] New platforms have evaluated Gateway API; legacy community Ingress-NGINX deployments have a migration plan.

## References

- [Kubernetes Ingress concept and API](https://kubernetes.io/docs/concepts/services-networking/ingress/)
- [Kubernetes Ingress controller options](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/)
- [Ingress-NGINX retirement announcement](https://kubernetes.io/blog/2025/11/11/ingress-nginx-retirement/)
- [Gateway API documentation](https://gateway-api.sigs.k8s.io/)
