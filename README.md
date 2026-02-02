# brouter-web-helm-chart

A Helm chart for deploying [brouter-web](https://github.com/nrenner/brouter-web) to Kubernetes.

## Prerequisites
- Kubernetes cluster (v1.24+ recommended)
- kubectl configured to talk to your cluster
- Helm v3.9+ installed
- Optional, if using Gateway API `HTTPRoute`: Gateway API CRDs and a compatible controller installed
- Depends on [BRouter](https://github.com/joe-akeem/brouter-helm-chart) for routing

## Chart contents
- Chart path: `./brouter-web`
- App version (default image tag): `0.18.1`
- Default container image: `joeakeem/brouter-web`
- Default service port: `80`

See `brouter-web/values.yaml` for all configurable parameters and their defaults.

## Quick start
Install into a namespace (create it if it doesn’t exist):

```bash
# From the repository root
helm upgrade --install brouter-web ./brouter-web \
  --namespace brouter --create-namespace
```

Wait for the deployment to become ready:
```bash
kubectl -n brouter get deploy,po,svc
```

## Accessing the service
- By default, a `ClusterIP` Service is created. Use `kubectl port-forward` for local access:
  ```bash
  kubectl -n brouter port-forward svc/brouter-web 8080:80
  # Open http://localhost:8080
  ```
- To expose it externally, enable either Ingress or Gateway API `HTTPRoute` (see below).

## Common customizations
Override values at install/upgrade time with `--set` or a values file (`-f my-values.yaml`).

Examples:

- Set a specific container image tag:
  ```bash
  helm upgrade --install brouter-web ./brouter-web \
    -n brouter --create-namespace \
    --set image.tag=0.18.1
  ```

- Change Service type to `LoadBalancer`:
  ```bash
  helm upgrade --install brouter-web ./brouter-web -n brouter \
    --set service.type=LoadBalancer
  ```

- Configure resources:
  ```yaml
  # my-values.yaml
  resources:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      cpu: 500m
      memory: 512Mi
  ```
  ```bash
  helm upgrade --install brouter-web ./brouter-web -n brouter -f my-values.yaml
  ```

## Enabling Ingress
Enable and configure Ingress (example for NGINX Ingress Controller):

```yaml
# ingress-values.yaml
ingress:
  enabled: true
  className: nginx
  hosts:
    - host: brouter.example.com
      paths:
        - path: /
          pathType: ImplementationSpecific
  tls:
    - secretName: brouter-tls
      hosts:
        - brouter.example.com
```

```bash
helm upgrade --install brouter-web ./brouter-web -n brouter -f ingress-values.yaml
```

## Exposing via Gateway API (HTTPRoute)
If your cluster has Gateway API installed and a controller configured, you can expose the service via `HTTPRoute`.

```yaml
# httproute-values.yaml
httpRoute:
  enabled: true
  parentRefs:
    - name: gateway   # must match an existing Gateway
      sectionName: http
      # namespace: default # uncomment if the Gateway is in a different namespace
  hostnames:
    - brouter.example.com
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
```

```bash
helm upgrade --install brouter-web ./brouter-web -n brouter -f httproute-values.yaml
```

Notes:
- Do not enable both `ingress.enabled` and `httpRoute.enabled` at the same time unless you specifically want both resources.
- `HTTPRoute` requires Gateway API CRDs and a controller such as Istio, Kuma, Envoy Gateway, or others.

## Autoscaling
Enable Horizontal Pod Autoscaling (HPA):
```yaml
# autoscaling-values.yaml
autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 75
```
```bash
helm upgrade --install brouter-web ./brouter-web -n brouter -f autoscaling-values.yaml
```

## Health probes
The chart defines default `livenessProbe` and `readinessProbe` hitting `/` on the `http` port. Customize via `values.yaml` if needed.

## Upgrading
```bash
# Apply new chart version or new values
helm upgrade brouter-web ./brouter-web -n brouter -f my-values.yaml
```

To see what will change before upgrading:
```bash
helm diff upgrade brouter-web ./brouter-web -n brouter -f my-values.yaml
# or render locally
helm template brouter-web ./brouter-web -n brouter -f my-values.yaml
```

## Uninstalling
```bash
helm uninstall brouter-web -n brouter
```
This removes the release but may leave persistent volumes or external DNS records that you created separately.

## Testing the release
Run the chart’s built-in test pod after installation:
```bash
helm test brouter-web -n brouter
```

## Configuration reference
Key values you may want to override (see `brouter-web/values.yaml` for the full list):
- `image.repository`, `image.tag`, `image.pullPolicy`
- `service.type`, `service.port`
- `ingress.enabled`, `ingress.className`, `ingress.hosts`, `ingress.tls`
- `httpRoute.enabled`, `httpRoute.parentRefs`, `httpRoute.hostnames`, `httpRoute.rules`
- `resources`, `nodeSelector`, `tolerations`, `affinity`
- `autoscaling.*`
- `podAnnotations`, `podLabels`, `securityContext`, `podSecurityContext`
- `volumes`, `volumeMounts`

## Notes
- The default image tag is the chart `appVersion` (`0.18.1`). Set `image.tag` to override.
- See `brouter-web/templates/NOTES.txt` for in-cluster access hints output by Helm after install.
