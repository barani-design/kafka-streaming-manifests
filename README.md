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
- Node labels and node capacity that satisfy the requirements in `Required Kubernetes Nodes And Resources`

Review and replace these values before applying the manifests in your own environment.

## Required Kubernetes Nodes And Resources

The default manifests assume a multi-node Kubernetes cluster with four workload buckets:

- `workload=kafka-broker` for Kafka broker/controller pods in [`kafka/kafka-kraft-pool.yaml`](kafka/kafka-kraft-pool.yaml) and broker pod affinity in [`kafka/kafka-kraft.yaml`](kafka/kafka-kraft.yaml)
- `workload=kafka-mgmt` for Strimzi management pods such as Cruise Control and the Entity Operator in [`kafka/kafka-kraft.yaml`](kafka/kafka-kraft.yaml)
- `workload=kafka-master` for Kafka UI in [`kafka/kafka-ui.yaml`](kafka/kafka-ui.yaml)
- `workload=kafka-stream` for Schema Registry, ksqlDB, Kafka Bridge, Debezium Connect, Camel Connect, and the ksqlDB CLI

### Default Node Count

Minimum for these manifests as written: `8` Kubernetes worker nodes.

- `5` nodes labeled `workload=kafka-broker`, one per default Kafka broker/controller replica
- `1` node labeled `workload=kafka-mgmt`
- `1` node labeled `workload=kafka-master`
- `1` node labeled `workload=kafka-stream`

Smaller non-production clusters can co-locate multiple labels on fewer nodes, but that reduces failure isolation and can make scheduling tighter during restarts, upgrades, or node loss. The `webhook-logger` example deployment does not have a node affinity rule, so it can run on any schedulable worker.

### Default Resource Footprint

Based on the manifests currently in this repository, the full stack requests about `7.75` CPU cores and `16.8 GiB` of memory, with limits around `10.2` CPU cores and `19.6 GiB` of memory.

Persistent storage requested by default is `120 Gi` total:

- `100 Gi` for the five Kafka broker PVCs in [`kafka/kafka-kraft-pool.yaml`](kafka/kafka-kraft-pool.yaml)
- `20 Gi` for ksqlDB state in [`ksqldb/pvc.yaml`](ksqldb/pvc.yaml)

Broken down by workload:

- Kafka broker pool: `5` replicas, each requesting and limiting `1` CPU and `2000Mi` memory, plus `20Gi` persistent storage per broker
- Management workloads: about `1` requested CPU core and `2.75 GiB` requested memory
- Stream workloads: about `1.55` requested CPU cores and `4 GiB` requested memory
- Kafka UI: `100m` CPU and `200Mi` memory requested

Treat those numbers as the minimum default footprint for this example repository, not as a production sizing recommendation. Real worker node capacity should be higher than the summed pod requests so the cluster still has room for Kubernetes system daemons, the Strimzi operator, ingress, cert-manager, monitoring, image pulls, filesystem cache, and disruption headroom.

### Example Node Labels

```sh
kubectl label node <broker-node-1> workload=kafka-broker
kubectl label node <broker-node-2> workload=kafka-broker
kubectl label node <broker-node-3> workload=kafka-broker
kubectl label node <broker-node-4> workload=kafka-broker
kubectl label node <broker-node-5> workload=kafka-broker
kubectl label node <mgmt-node> workload=kafka-mgmt
kubectl label node <ui-node> workload=kafka-master
kubectl label node <stream-node> workload=kafka-stream
```

### Kafka Sizing Notes

The [Apache Kafka hardware guidance](https://kafka.apache.org/42/operations/hardware-and-os/) does not give a single universal node size for every deployment, but it highlights a few rules that matter here:

- memory sizing should account for active reader/writer buffering and operating system page cache; Kafka suggests a rough estimate of `write_throughput * 30` for 30 seconds of buffering
- disk throughput is often the primary bottleneck, so storage performance matters as much as raw CPU and RAM
- Kafka data should not share the same disks used for OS or application log activity
- broker processes should start with at least `100000` available file descriptors

Strimzi's resource guidance in the [configuration documentation](https://strimzi.io/docs/operators/latest/configuring) also applies here: resource requests should be high enough for stable performance, and pods will remain unscheduled if cluster nodes do not have enough free CPU or memory to satisfy those requests.

If you change throughput targets, retention, partition counts, replication factor, storage classes, or the number of enabled services, resize the node count and per-node capacity before applying these manifests.

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
