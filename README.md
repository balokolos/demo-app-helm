# demo-app Helm Chart

A sample Helm chart that deploys a simple demo application into its own namespace `demo-app`.

This chart demonstrates the following Kubernetes resources:

- `Namespace` (`demo-app`)
- `ServiceAccount`
- `Deployment` (creates a ReplicaSet)
- `ConfigMap`
- `Secret`
- `PersistentVolumeClaim` using Longhorn storage
- `Service` of type `LoadBalancer`
- `GatewayClass`
- `Gateway`
- `HTTPRoute`

## Chart layout

- `Chart.yaml` - chart metadata
- `values.yaml` - default configuration values
- `templates/` - Kubernetes manifest templates
  - `namespace.yaml`
  - `serviceaccount.yaml`
  - `deployment.yaml`
  - `service.yaml`
  - `configmap.yaml`
  - `secret.yaml`
  - `pvc.yaml`
  - `gatewayclass.yaml`
  - `gateway.yaml`
  - `httproute.yaml`

## Purpose

This chart creates a small sample workload around an `nginx` container, with environment variables sourced from a `ConfigMap` and `Secret`, and a persistent volume mounted from a Longhorn-backed PVC.

It also demonstrates Gateway API integration by creating a `Gateway` and `HTTPRoute` with a hostname of `demo-app.local`, using the existing Istio GatewayClass by default.

## Default behavior

By default, the chart deploys:

- `replicaCount: 1`
- `image.repository: nginx`
- `service.type: LoadBalancer`
- `service.port: 80`
- `targetNamespace: demo-app`
- `pvc.storageClassName: longhorn`
- `pvc.storage: 1Gi`
- `httpRoute.enabled: true`
- `gateway.enabled: true`
- `gatewayClass.enabled: false`

## Key values

### `targetNamespace`

The chart creates and installs into the `demo-app` namespace.

### `image`

- `repository`: `nginx`
- `tag`: defaults to `.Chart.AppVersion`
- `pullPolicy`: `IfNotPresent`

### `service`

- `type`: `LoadBalancer`
- `port`: `80`

### `httpRoute`

- `enabled`: `true`
- `hostnames`: `demo-app.local`
- `rules`: one prefix rule matching `/`
- `parentRefs`: points to `demo-app-gateway` in namespace `demo-app`

### `gatewayClass`

- `enabled`: `false`
- `name`: `istio`
- `controller`: `istio.io/gateway-controller`

> Note: By default this chart uses the existing Istio GatewayClass named `istio`. If you want Helm to create a custom GatewayClass, set `gatewayClass.enabled=true` and update `gatewayClass.name` and `gatewayClass.controller`.

### `gateway`

- `enabled`: `true`
- `name`: `demo-app-gateway`
- listener on port `80`, protocol `HTTP`, hostname `demo-app.local`

### `pvc`

- `enabled`: `true`
- `storageClassName`: `longhorn`
- `accessModes`: `ReadWriteOnce`
- `storage`: `1Gi`

### `config`

- `appName`: `demo-app`
- `appMode`: `demo`
- `logLevel`: `info`

### `secret`

- `enabled`: `true`
- `password`: `demo-password`

## What is mounted

The chart mounts the Longhorn-backed PVC at `/usr/share/nginx/html` inside the `nginx` container.

This means the persistent volume is used as the application content root for the sample `nginx` workload.

## Deploying the chart

From the chart root:

```bash
cd /home/hilmi/balok/playground/helm/demo-app
helm upgrade --install demo-app . --namespace demo-app --create-namespace
```

This will install or upgrade the `demo-app` release into the `demo-app` namespace.

## Inspecting the deployment

```bash
kubectl get all -n demo-app
kubectl get pvc -n demo-app
kubectl get gatewayclass
kubectl get gateway -n demo-app
kubectl get httproute -n demo-app
```

## Customizing values

To override values, use `--set` or a custom values file.

Example:

```bash
helm upgrade --install demo-app . \
  --namespace demo-app --create-namespace \
  --set replicaCount=3 \
  --set service.type=ClusterIP \
  --set pvc.storage=2Gi \
  --set gatewayClass.enabled=true \
  --set gatewayClass.name=istio \
  --set gatewayClass.controller=istio.io/gateway-controller
```

## Notes and caveats

- By default, this chart does not create a custom `GatewayClass`; it uses the existing Istio GatewayClass named `istio`. If your cluster uses a different GatewayClass or you want Helm to create one, set `gatewayClass.enabled=true` and update `gatewayClass.name` and `gatewayClass.controller` in `values.yaml`.
- The `HTTPRoute` host is `demo-app.local`; you may need to map this hostname locally or update it to a valid DNS name for your environment.
- If Longhorn is not available, update `pvc.storageClassName` to a supported storage class.
- This chart is intended as a sample template and can be extended with application-specific readiness probes, resources, or init containers.

## Cleanup

To remove the release and namespace:

```bash
helm uninstall demo-app --namespace demo-app
kubectl delete namespace demo-app
```

## GitHub Actions workflow

This repository includes a GitHub workflow at `.github/workflows/helm-package.yml` that packages and optionally releases the Helm chart.

What it does:

- runs on `push` to `main` and on manual `workflow_dispatch`
- installs Helm and validates the chart with `helm lint`
- builds chart dependencies and packages the chart into `packaged/*.tgz`
- uploads the packaged chart as a workflow artifact
- when a release is requested, it additionally:
  - authenticates to GitHub Container Registry using `RELEASE_PAT`
  - pushes the chart package to `oci://ghcr.io/balokolos/charts`
  - creates or updates a GitHub release with the packaged chart attached

Release conditions:

- automatic release on `main` when the commit message contains `release=true`
- manual release when the workflow is dispatched with input `release: true`

## Package

hosted in github.
[https://github.com/users/balokolos/packages/container/package/charts%2Fdemo-app](https://github.com/users/balokolos/packages/container/package/charts%2Fdemo-app)
