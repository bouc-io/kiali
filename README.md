# Kiali Installation and Configuration

Overall Kiali installation and configuration for all environments (i.e. local, sandbox and production).

## Prerequisites

The following configuration is based on Docker and Docker-desktop setup, leveraging its internal, local kubernetes cluster (usually named "docker-desktop").

Istio must be installed in the specific namespace called "istio-system". Refer to the Istio installation, required.

To deploy Kiali manually via Helm, execute following command:

```shell
helm repo add kiali https://kiali.org/helm-charts/
helm repo update
```

## Local

To install Kiali locally, execute the following command:

```shell
helm upgrade -install --namespace kiali --create-namespace --set auth.strategy="anonymous" kiali-server kiali-server -f base.values.yaml -f lcl.values.yaml
```

Prior to access the kiali dashboard, add the following entry to the /etc/hosts file:

127.0.0.1 kiali.docker.internal

Access Kiali at: http://kiali.docker.internal/

## Sandbox

The setup for Sandbox mirrors Local but uses `sbnx.values.yaml`.


### References

Refer to:
- https://kiali.io/documentation/latest/installation-guide/#_helm_chart
- https://github.com/kiali/helm-charts/blob/master/kiali-operator/values.yaml

## FluxCD GitOps

Kiali is delivered by FluxCD. The Flux repo reconciles manifests through environment-specific Kustomizations which reference the values files located in this directory.

- `base.values.yaml` – Common configuration logic (Ingress, External Services).
- `lcl.values.yaml` – Local-specific overrides (replicas=1, anonymous auth).
- `sbnx.values.yaml` – Sandbox specific overrides, including configuration (e.g. OIDC authentication, replicas).

Flux wires them via `configMapGenerator` in the cluster's config Kustomization:
```yaml
  - name: kiali-base-values
    namespace: kiali
    files:
      - base.values.yaml=../../components/kiali/base.values.yaml
  - name: kiali-level-values
    namespace: kiali
    files:
      - lcl.values.yaml=../../components/kiali/lcl.values.yaml
```

The OIDC client secret needs no human step at all. Keycloak is the source of truth:
the `keycloak-kiali-sync` Job in `infrastructure/keycloak` (a `post-install,post-upgrade`
Helm hook in `extraDeploy`) creates the master-realm `kiali-client` if it is absent,
reads back its real secret with `kcadm.sh`, and writes both consumers directly:

| Secret | Key | Namespace | Consumer |
|---|---|---|---|
| `kiali` | `oidc-secret` | `kiali` | this chart, via `deployment.secret_name` |
| `grafana-oauth` | `client-secret` | `grafana` | Grafana, which deliberately reuses `kiali-client` |

The Job then restarts both Deployments, because each reads the value only once at
start. This replaces the previous External Secrets delivery (`boucio-kiali-oidc`,
`boucio-grafana-oauth`), which needed a value pre-seeded before Keycloak even existed
and failed silently when it held a placeholder.

To force a re-sync (for example after rotating the client secret in Keycloak), re-run
the idempotent Job by forcing a Keycloak upgrade:
```shell
flux reconcile helmrelease keycloak-helmrelease -n keycloak --force
```

For a standalone `helm install` outside this GitOps setup, where no Keycloak Job runs,
create the Secret manually:
```shell
kubectl create secret generic kiali \
  --from-literal=oidc-secret='<YOUR_CLIENT_SECRET>' \
  -n kiali
```

## License

[Elastic License 2.0](./LICENSE) — source-available; not OSI open source.
