# gitops-manifest-k8s

GitOps manifests for YAS (Yet Another Shop), managed with Kustomize and synced by ArgoCD.

## Structure

- `base/`: shared Deployment, Service, and ServiceAccount manifests for 19 YAS services.
- `environments/dev/`: deploys the base manifests into the `dev` namespace with `latest` image tags by default.
- `environments/staging/`: deploys into the `staging` namespace with release tags such as `v1.0.0`.
- `environments/developer-build/`: deploys into `developer-build` and patches all Services to `NodePort`.

## Services

| Service | Port | Image |
| --- | ---: | --- |
| media | 8083 | `bingsu1103/media:latest` |
| product | 8080 | `bingsu1103/product:latest` |
| order | 8085 | `bingsu1103/order:latest` |
| inventory | 8090 | `bingsu1103/inventory:latest` |
| payment | 8081 | `bingsu1103/payment:latest` |
| promotion | 8092 | `bingsu1103/promotion:latest` |
| rating | 8089 | `bingsu1103/rating:latest` |
| delivery | 8080 | `bingsu1103/delivery:latest` |
| sampledata | 8094 | `bingsu1103/sampledata:latest` |
| recommendation | 8095 | `bingsu1103/recommendation:latest` |
| customer | 8088 | `bingsu1103/customer:latest` |
| location | 8086 | `bingsu1103/location:latest` |
| cart | 8084 | `bingsu1103/cart:latest` |
| tax | 8091 | `bingsu1103/tax:latest` |
| search | 8092 | `bingsu1103/search:latest` |
| webhook | 8092 | `bingsu1103/webhook:latest` |
| backoffice-bff | 8087 | `bingsu1103/backoffice-bff:latest` |
| storefront-bff | 8087 | `bingsu1103/storefront-bff:latest` |
| payment-paypal | 8093 | `bingsu1103/payment-paypal:latest` |

## Validate

```sh
kubectl kustomize environments/dev
kubectl kustomize environments/staging
kubectl kustomize environments/developer-build
```

Expected checks:

```sh
kubectl kustomize environments/dev | grep "kind: Deployment" | wc -l
kubectl kustomize environments/dev | grep "kind: ServiceAccount" | wc -l
kubectl kustomize environments/dev | grep "serviceAccountName:" | wc -l
kubectl kustomize environments/developer-build | grep "type: NodePort" | wc -l
```

Each command should return `19` for the corresponding resource count.

## Update Image Tags

Jenkins can update image tags with Kustomize:

```sh
cd environments/dev
kustomize edit set image bingsu1103/tax=bingsu1103/tax:abc1234
```

For `developer-build`, Jenkins should update only the services being tested and leave unchanged services on `latest`.

## Add A New Service

1. Create `base/<service-name>/`.
2. Add `deployment.yaml`, `service.yaml`, and `serviceaccount.yaml`.
3. Add those files to `base/kustomization.yaml`.
4. Add the image entry to all environment kustomizations.
5. Run the validate commands above.
