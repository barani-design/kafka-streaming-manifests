# Kafka Platform Manifests

This repository contains Kubernetes manifests for a Kafka-centered platform built on Strimzi and related services.

It is intended to be published as a single open-source repository with component directories, not as six unrelated projects.

## What Is Included

- [`kafka/README.md`](kafka/README.md): Strimzi Kafka KRaft cluster, node pool, Kafka user, rebalance, and Kafka UI.
- [`schema-registry/README.md`](schema-registry/README.md): Confluent Schema Registry deployment, service, and ingress.
- [`ksqldb/README.md`](ksqldb/README.md): ksqlDB server, CLI, service, PVC, and ingress.
- [`kafka-bridge/README.md`](kafka-bridge/README.md): Strimzi Kafka Bridge and ingress.
- [`debezium-connect/README.md`](debezium-connect/README.md): Strimzi Kafka Connect with Debezium PostgreSQL source configuration and RBAC.
- [`camel-connect/README.md`](camel-connect/README.md): Strimzi Kafka Connect with Camel Netty HTTP webhook ingestion, topics, services, ingress, and a PlantUML diagram source.

## What Is Not Included

This repository contains infrastructure manifests and example component wiring. It does not contain the full checked-in business-specific stream-processing logic that would transform raw events into final domain outputs.

In practice, that means:

- Kafka topics and ingestion paths are defined here.
- Supporting services such as Schema Registry and ksqlDB are deployed here.
- Example CDC and webhook ingestion components are deployed here.
- The actual production stream-processing jobs, queries, or application code are not part of this repository.

## Repository Layout

```text
kafka/
schema-registry/
ksqldb/
kafka-bridge/
debezium-connect/
camel-connect/
```

Each directory contains manifests plus a component-specific `README.md`.

## Deployment Order

Use this order as the baseline:

1. Deploy [`kafka`](kafka/README.md).
2. Deploy optional platform services: [`schema-registry`](schema-registry/README.md), [`ksqldb`](ksqldb/README.md), and [`kafka-bridge`](kafka-bridge/README.md).
3. Deploy integration components: [`debezium-connect`](debezium-connect/README.md) and [`camel-connect`](camel-connect/README.md).

## Shared Assumptions

Most manifests currently assume:

- Namespace: `kafka`
- Strimzi Kafka cluster name: `kafka-kraft`
- Kafka bootstrap service: `kafka-kraft-kafka-bootstrap.kafka.svc:9092`
- SCRAM Secret name: `kafka-kraft-user`
- Placeholder ingress hosts under `example.com`
- Node labels that satisfy `nodeAffinity` rules

Review and replace these values before applying the manifests in your own environment.

## Configuration You Must Replace

Before using these manifests outside this repository's example setup, update:

- ingress hostnames
- image registry placeholders such as `ghcr.io/example/...`
- Secret names and Secret contents where needed
- storage classes and storage sizes
- node labels and affinity rules
- replication factors and sizing defaults

## Validation

At minimum:

```sh
python3 -c "import pathlib,yaml; [list(yaml.safe_load_all(p.read_text())) for p in pathlib.Path('.').glob('**/*.yaml')]; print('yaml parse ok')"
```

For release-quality validation, test against a Kubernetes cluster with Strimzi CRDs installed:

```sh
kubectl apply --dry-run=server -f kafka/
kubectl apply --dry-run=server -f schema-registry/
kubectl apply --dry-run=server -f ksqldb/
kubectl apply --dry-run=server -f kafka-bridge/
kubectl apply --dry-run=server -f debezium-connect/
kubectl apply --dry-run=server -f camel-connect/
```

## Security Notes

- This repository is designed to avoid committed live secrets.
- Secrets referenced by the manifests must be created separately in your cluster.
- Some manifests use `SASL_PLAINTEXT` as an internal example. Switch to TLS or `SASL_SSL` if your environment requires encryption in transit.

## Diagram Note

The Camel Connect topology is currently tracked as PlantUML source in [`camel-connect/images/camel-connect-diagram.puml`](camel-connect/images/camel-connect-diagram.puml). The `.puml` file is the canonical diagram source for this repository.
