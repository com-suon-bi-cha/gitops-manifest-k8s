# Dev Infrastructure Helm Values

This directory stores the Helm values used to recreate infrastructure
dependencies in the `dev` namespace manually.

The running cluster was originally bootstrapped manually with Helm. These files
capture the same release names and chart versions so the infra can be recreated
without guessing values again.

## Releases

| Release | Chart | Version |
| --- | --- | --- |
| postgres | bitnami/postgresql | 18.7.6 |
| redis | bitnami/redis | 27.0.10 |
| kafka | bitnami/kafka | 32.4.3 |
| keycloak | bitnami/keycloak | 25.2.0 |
| elasticsearch | bitnami/elasticsearch | 22.1.6 |

## Notes

- `postgres-init` must exist before a fresh PostgreSQL install so the service
  databases are created on first boot.
- Kafka values reference the existing `kafka-user-passwords` and `kafka-kraft`
  secrets to avoid changing generated passwords and the KRaft cluster id.
- The current project uses simple demo credentials for dev. For a production
  setup, move credentials to SealedSecret, SOPS, or ExternalSecrets.

## Recreate

```sh
kubectl apply -k infra/dev/postgres-init

helm upgrade --install postgres bitnami/postgresql \
  -n dev --create-namespace \
  -f infra/dev/helm-values/postgres-values.yaml

helm upgrade --install redis bitnami/redis \
  -n dev --create-namespace \
  -f infra/dev/helm-values/redis-values.yaml

helm upgrade --install kafka bitnami/kafka \
  -n dev --create-namespace \
  -f infra/dev/helm-values/kafka-values.yaml

helm upgrade --install keycloak bitnami/keycloak \
  -n dev --create-namespace \
  -f infra/dev/helm-values/keycloak-values.yaml

helm upgrade --install elasticsearch bitnami/elasticsearch \
  -n dev --create-namespace \
  -f infra/dev/helm-values/elasticsearch-values.yaml
```
