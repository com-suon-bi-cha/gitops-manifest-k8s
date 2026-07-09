# YAS Infrastructure Recreate Guide

This guide recreates the YAS infrastructure dependencies with Helm values stored
in this GitOps repo. It uses manual Helm commands, not ArgoCD Applications, for
infrastructure.

## Scope

This guide covers namespace-level YAS dependencies:

- PostgreSQL
- Redis
- Kafka
- Keycloak
- Elasticsearch
- `postgres-init` ConfigMap
- YAS app workloads from `environments/dev` and `environments/staging`

This guide does not install cluster-level components. Install these first if the
cluster is new:

- Kubernetes/K3s
- Istio
- ArgoCD, if you want ArgoCD to sync YAS app workloads
- Kiali/Prometheus/Grafana, if you need service mesh screenshots
- Observability stack, if you need metrics/logs/traces dashboards

This repo also contains cluster-level manifests in `cluster/`. Apply them once
per cluster before deploying namespace workloads.

## Prerequisites

```sh
kubectl cluster-info
helm version
kubectl get storageclass
```

## Cluster Bootstrap

Run this once for a new cluster:

```sh
kubectl apply -k cluster
kubectl rollout status ds/node-sysctl -n kube-system --timeout=120s
kubectl rollout restart deploy/coredns -n kube-system
kubectl rollout status deploy/coredns -n kube-system --timeout=120s
```

`cluster/node-sysctl.yaml` raises node file/inotify limits. This is required to
avoid resource pressure issues when Elasticsearch, Istio sidecars, and many Java
pods run on the same small node.

`cluster/coredns-custom.yaml` lets pods resolve internal YAS hostnames such as
`identity.dev.yas.local.com` and `identity.staging.yas.local.com`. If this file
is moved to another cluster, update the IP in `cluster/coredns-custom.yaml` to
the target worker node internal IP before applying:

```sh
kubectl get nodes -o wide
```

The browser machine still needs its own `/etc/hosts` entries. CoreDNS only
affects DNS inside the Kubernetes cluster.

Add the Bitnami repository:

```sh
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

If the cluster uses Istio, label the namespaces before deploying workloads:

```sh
kubectl create namespace dev --dry-run=client -o yaml | kubectl apply -f -
kubectl create namespace staging --dry-run=client -o yaml | kubectl apply -f -
kubectl create namespace developer-build --dry-run=client -o yaml | kubectl apply -f -

kubectl label namespace dev istio-injection=enabled --overwrite
kubectl label namespace staging istio-injection=enabled --overwrite
kubectl label namespace developer-build istio-injection=enabled --overwrite
```

## Fresh Cluster Install

Use this path when the target namespace has no existing PVCs or Helm release
history.

### Dev Infra

```sh
kubectl apply -k infra/dev/postgres-init

helm upgrade --install postgres bitnami/postgresql \
  --version 18.7.6 \
  -n dev --create-namespace \
  -f infra/dev/helm-values/postgres-values.yaml

helm upgrade --install redis bitnami/redis \
  --version 27.0.10 \
  -n dev --create-namespace \
  -f infra/dev/helm-values/redis-values.yaml

helm upgrade --install kafka bitnami/kafka \
  --version 32.4.3 \
  -n dev --create-namespace \
  -f infra/dev/helm-values/kafka-values.yaml

helm upgrade --install keycloak bitnami/keycloak \
  --version 25.2.0 \
  -n dev --create-namespace \
  -f infra/dev/helm-values/keycloak-values.yaml

helm upgrade --install elasticsearch bitnami/elasticsearch \
  --version 22.1.6 \
  -n dev --create-namespace \
  -f infra/dev/helm-values/elasticsearch-values.yaml
```

Wait until infra is ready:

```sh
kubectl rollout status sts/postgres-postgresql -n dev --timeout=300s
kubectl rollout status sts/redis-master -n dev --timeout=300s
kubectl rollout status sts/kafka-controller -n dev --timeout=300s
kubectl rollout status sts/keycloak -n dev --timeout=300s
kubectl rollout status sts/elasticsearch-master -n dev --timeout=300s
kubectl rollout status sts/elasticsearch-data -n dev --timeout=300s
kubectl rollout status sts/elasticsearch-coordinating -n dev --timeout=300s
```

### Staging Infra

```sh
kubectl apply -k infra/staging/postgres-init

helm upgrade --install postgres bitnami/postgresql \
  --version 18.7.6 \
  -n staging --create-namespace \
  -f infra/staging/helm-values/postgres-values.yaml

helm upgrade --install redis bitnami/redis \
  --version 27.0.10 \
  -n staging --create-namespace \
  -f infra/staging/helm-values/redis-values.yaml

helm upgrade --install kafka bitnami/kafka \
  --version 32.4.3 \
  -n staging --create-namespace \
  -f infra/staging/helm-values/kafka-values.yaml

helm upgrade --install keycloak bitnami/keycloak \
  --version 25.2.0 \
  -n staging --create-namespace \
  -f infra/staging/helm-values/keycloak-values.yaml

helm upgrade --install elasticsearch bitnami/elasticsearch \
  --version 22.1.6 \
  -n staging --create-namespace \
  -f infra/staging/helm-values/elasticsearch-values.yaml
```

Wait until infra is ready:

```sh
kubectl rollout status sts/postgres-postgresql -n staging --timeout=300s
kubectl rollout status sts/redis-master -n staging --timeout=300s
kubectl rollout status sts/kafka-controller -n staging --timeout=300s
kubectl rollout status sts/keycloak -n staging --timeout=300s
kubectl rollout status sts/elasticsearch-master -n staging --timeout=300s
kubectl rollout status sts/elasticsearch-data -n staging --timeout=300s
kubectl rollout status sts/elasticsearch-coordinating -n staging --timeout=300s
```

## Deploy YAS Applications

Use ArgoCD for app workloads:

```sh
kubectl apply -f argocd/yas-dev-app.yaml
kubectl apply -f argocd/yas-staging-app.yaml
```

Or apply manually with Kustomize:

```sh
kubectl apply -k environments/dev
kubectl apply -k environments/staging
```

Developer-build uses dev infrastructure:

```sh
kubectl apply -k environments/developer-build
```

## Manual Items Not Stored In These Helm Values

The Helm values recreate infrastructure pods and services, but a completely
fresh cluster still needs these items checked:

- Install Istio and ArgoCD before applying the YAS application manifests.
- Add local `/etc/hosts` entries on the browser machine.
- Import or recreate the Keycloak `Yas` realm, clients, redirect URIs, roles,
  and demo users if Keycloak/PostgreSQL data is empty. Existing PVCs keep this
  state; a fresh PostgreSQL database does not.
- Run or restart `sampledata` after PostgreSQL, media, product, and search are
  ready so demo products and media records are populated.
- Build and push Docker images before updating image tags in GitOps.

## Verify

```sh
kubectl get pods -n dev
kubectl get pods -n staging
kubectl get sts,svc -n dev | grep -E 'postgres|redis|kafka|keycloak|elastic'
kubectl get sts,svc -n staging | grep -E 'postgres|redis|kafka|keycloak|elastic'
```

If app pods started before infra was ready, restart them:

```sh
kubectl rollout restart deploy -n dev
kubectl rollout restart deploy -n staging
```

## Local Hosts

Because the project uses local hostnames instead of public DNS, add these
records on the machine that opens the browser:

```text
<WORKER_IP> storefront.dev.yas.local.com
<WORKER_IP> backoffice.dev.yas.local.com
<WORKER_IP> swagger.dev.yas.local.com
<WORKER_IP> identity.dev.yas.local.com
<WORKER_IP> storefront.staging.yas.local.com
<WORKER_IP> identity.staging.yas.local.com
```

Default URLs:

```text
http://storefront.dev.yas.local.com:31255
http://backoffice.dev.yas.local.com:31255
http://swagger.dev.yas.local.com:31255/swagger-ui/index.html
http://storefront.staging.yas.local.com:31255
```

## Restore Existing Namespace With PVCs

If you only deleted StatefulSets/Services/ConfigMaps but kept PVCs, be careful
with Kafka. Kafka stores the KRaft cluster id on disk and also in Kubernetes
Secret `kafka-kraft`.

Do not delete these when doing a partial restore:

```text
kafka-kraft
kafka-user-passwords
postgres-postgresql
keycloak
keycloak-externaldb
data-kafka-controller-0
data-postgres-postgresql-0
redis-data-redis-master-0
data-elasticsearch-master-0
data-elasticsearch-data-0
```

If those Kafka secrets are deleted while the old Kafka PVC is retained, Kafka
may fail because a newly generated KRaft cluster id will not match the existing
disk metadata. In that case, restore the old secrets from backup or Helm release
history before starting Kafka.

## Scale Down To Save Resources

Scale app deployments:

```sh
kubectl scale deploy --all -n staging --replicas=0
```

Scale staging infra:

```sh
kubectl scale sts postgres-postgresql redis-master kafka-controller keycloak \
  elasticsearch-master elasticsearch-data elasticsearch-coordinating \
  -n staging --replicas=0
```

Scale back up:

```sh
kubectl scale sts postgres-postgresql redis-master kafka-controller keycloak \
  elasticsearch-master elasticsearch-data elasticsearch-coordinating \
  -n staging --replicas=1

kubectl scale deploy --all -n staging --replicas=1
```

Restart app deployments after infra is ready:

```sh
kubectl rollout restart deploy -n staging
```

## Validate Values Without Applying

```sh
kubectl kustomize infra/dev/postgres-init
kubectl kustomize infra/staging/postgres-init

helm template postgres bitnami/postgresql --version 18.7.6 -n dev -f infra/dev/helm-values/postgres-values.yaml
helm template redis bitnami/redis --version 27.0.10 -n dev -f infra/dev/helm-values/redis-values.yaml
helm template kafka bitnami/kafka --version 32.4.3 -n dev -f infra/dev/helm-values/kafka-values.yaml
helm template keycloak bitnami/keycloak --version 25.2.0 -n dev -f infra/dev/helm-values/keycloak-values.yaml
helm template elasticsearch bitnami/elasticsearch --version 22.1.6 -n dev -f infra/dev/helm-values/elasticsearch-values.yaml
```
