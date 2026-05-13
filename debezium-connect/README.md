# Debezium Connect

This directory contains raw Kubernetes manifests for a Strimzi `KafkaConnect` cluster with the Debezium PostgreSQL connector, topic definitions, and the RBAC required to read database credentials from a Kubernetes Secret.

Return to the [root README](../README.md) for the overall repository layout and deployment order.

## Included manifests

- `connect.yaml`: `KafkaConnect` cluster with the Debezium PostgreSQL plugin.
- `postgres-source-connector.yaml`: example PostgreSQL source connector.
- `topics.yaml`: example Kafka topics for captured data and downstream reference data.
- `rbac-role-debezium-secret.yaml`: least-privilege RBAC that allows the Connect service account to read one Secret.

## Prerequisites

- Strimzi Kafka Operator installed with connector resources enabled.
- A reachable Kafka cluster and a SCRAM Secret named `kafka-kraft-user`.
- A PostgreSQL database configured for logical replication.
- The `kafka` namespace.
- A container registry path you can push the built Kafka Connect image to.

## Required customizations

Review these values before deployment:

- Output image in `connect.yaml` (`ghcr.io/example/debezium-connect-cluster:latest` is a placeholder).
- Database hostname, database name, publication, slot, and table list in `postgres-source-connector.yaml`.
- Topic names, partitions, and retention in `topics.yaml`.
- Node labels and resource limits in `connect.yaml`.

Create the database credentials Secret before applying the connector:

```sh
kubectl -n kafka create secret generic postgres-source-credentials \
  --from-literal=DB_USERNAME=postgres \
  --from-literal=DB_PASSWORD=change-me
```

## PostgreSQL requirements

The example connector uses:

- `plugin.name: pgoutput`
- `publication.name: sample_publication`
- `slot.name: sample_slot`

Create matching PostgreSQL objects and grant the connector user permission to use logical replication before enabling the connector.

## Deployment order

```sh
kubectl apply -f topics.yaml
kubectl apply -f rbac-role-debezium-secret.yaml
kubectl apply -f connect.yaml
kubectl apply -f postgres-source-connector.yaml
```

Wait for the `KafkaConnect` resource to become ready before applying the connector if your cluster is slow to build the image.

## Validation

```sh
kubectl get kafkaconnect,kafkaconnector -n kafka
kubectl logs deployment/debezium-connect-cluster-connect -n kafka
kubectl describe kafkaconnector debezium.source.jdbc.pgsql.sampledb.raw.v1 -n kafka
```

## Security notes

- `rbac-role-debezium-secret.yaml` intentionally limits Secret access to `postgres-source-credentials`.
- Replace all placeholder database values before production use.
- `snapshot.mode: always` is included as an example and may not be appropriate for large or production databases.

## Uninstall

```sh
kubectl delete -f postgres-source-connector.yaml
kubectl delete -f connect.yaml
kubectl delete -f rbac-role-debezium-secret.yaml
kubectl delete -f topics.yaml
```

## References

- [Debezium PostgreSQL connector documentation](https://debezium.io/documentation/reference/stable/connectors/postgresql.html)
- [Strimzi Kafka Connect builds](https://strimzi.io/docs/operators/latest/configuring.html#assembly-kafka-connect-build-str)
