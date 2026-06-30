# gitops-manifest-k8s

GitOps manifests for YAS, managed with Kustomize and synced by ArgoCD.

## Structure

- ase/: shared Deployment, Service, and ServiceAccount manifests for 19 YAS services.
- environments/dev/: deploys the base manifests into the dev namespace with latest image tags by default.
- environments/staging/: deploys into the staging namespace with release tags such as 1.0.0.
- environments/developer-build/: deploys into developer-build and patches all Services to NodePort.

## Services

| Service | Port | Image |
| --- | ---: | --- || media | 8083 | ingsu1103/media:latest |
| product | 8080 | ingsu1103/product:latest |
| order | 8085 | ingsu1103/order:latest |
| inventory | 8090 | ingsu1103/inventory:latest |
| payment | 8081 | ingsu1103/payment:latest |
| promotion | 8092 | ingsu1103/promotion:latest |
| rating | 8089 | ingsu1103/rating:latest |
| delivery | 8080 | ingsu1103/delivery:latest |
| sampledata | 8094 | ingsu1103/sampledata:latest |
| recommendation | 8095 | ingsu1103/recommendation:latest |
| customer | 8088 | ingsu1103/customer:latest |
| location | 8086 | ingsu1103/location:latest |
| cart | 8084 | ingsu1103/cart:latest |
| tax | 8091 | ingsu1103/tax:latest |
| search | 8092 | ingsu1103/search:latest |
| webhook | 8092 | ingsu1103/webhook:latest |
| backoffice-bff | 8087 | ingsu1103/backoffice-bff:latest |
| storefront-bff | 8087 | ingsu1103/storefront-bff:latest |
| payment-paypal | 8093 | ingsu1103/payment-paypal:latest |

## Validate

`sh
kubectl kustomize environments/dev
kubectl kustomize environments/staging
kubectl kustomize environments/developer-build
`

Expected checks:

`sh
kubectl kustomize environments/dev | grep "kind: Deployment" | wc -l
kubectl kustomize environments/dev | grep "kind: ServiceAccount" | wc -l
kubectl kustomize environments/developer-build | grep "type: NodePort" | wc -l
`

Each command should return 19 for the corresponding resource count.

## Update Image Tags

Jenkins can update image tags with Kustomize:

`sh
cd environments/dev
kustomize edit set image bingsu1103/tax=bingsu1103/tax:abc1234
`

For developer-build, Jenkins should update only the services being tested and leave unchanged services on latest.
