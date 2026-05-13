# Kafka KRaft on Kubernetes

This directory contains raw Kubernetes manifests for a Strimzi-managed Apache Kafka cluster running in KRaft mode, plus an optional Kafka UI deployment.

Return to the [root README](../README.md) for the overall repository layout and deployment order.

## Included manifests

- `kafka-kraft-pool.yaml`: `KafkaNodePool` for combined broker/controller nodes.
- `kafka-kraft.yaml`: `Kafka` custom resource with Cruise Control and entity operators.
- `kafka-user.yaml`: SCRAM user used by the other components in this repository.
- `kafka-rebalance.yaml`: `KafkaRebalance` resource for Cruise Control.
- `kafka-ui.yaml`: Kafka UI deployment, service, and ingress.

## Prerequisites

- Kubernetes cluster with a default storage class or a storage class that matches `kafka-kraft-pool.yaml`.
- Strimzi Kafka Operator installed with support for KRaft and node pools.
- Nodes labeled to satisfy the `nodeAffinity` rules in the manifests.
- The `kafka` namespace.
- An ingress controller and cert-manager if you want to expose Kafka UI.

## Required customizations

Review these settings before applying the manifests:

- Storage class in `kafka-kraft-pool.yaml` (`standard` by default).
- Broker, controller, and service node labels in the `nodeAffinity` sections.
- Cluster size, storage size, and replication factors.
- Kafka UI ingress host in `kafka-ui.yaml` (`kafka-ui.example.com` by default).
- Kafka UI login secret name `kafka-ui-auth`.

Create the Kafka UI login secret before deploying `kafka-ui.yaml`:

```sh
kubectl -n kafka create secret generic kafka-ui-auth \
  --from-literal=username=admin \
  --from-literal=password=change-me
```

## Deployment order

```sh
kubectl create namespace kafka
kubectl apply -f kafka-kraft-pool.yaml
kubectl apply -f kafka-kraft.yaml
kubectl apply -f kafka-user.yaml
kubectl apply -f kafka-rebalance.yaml
kubectl apply -f kafka-ui.yaml
```

## Validation

```sh
kubectl get kafka,kafkanodepool,kafkauser,kafkarebalance -n kafka
kubectl get pods,svc,ingress -n kafka
kubectl describe kafka kafka-kraft -n kafka
```

If Kafka UI is enabled, open `https://kafka-ui.example.com` after updating the ingress host and DNS.

## Security notes

- `kafka-user.yaml` grants broad topic and consumer-group permissions. Restrict ACLs before using this in production.
- Several components in this repository use `SASL_PLAINTEXT` when connecting inside the cluster. If your environment requires encryption in transit, switch those clients to TLS or `SASL_SSL`.
- The Kafka UI credentials are intentionally sourced from a Kubernetes Secret and should not be committed to git.

## Uninstall

```sh
kubectl delete -f kafka-ui.yaml
kubectl delete -f kafka-rebalance.yaml
kubectl delete -f kafka-user.yaml
kubectl delete -f kafka-kraft.yaml
kubectl delete -f kafka-kraft-pool.yaml
```

## References

- [Strimzi Kafka Operator](https://github.com/strimzi/strimzi-kafka-operator)
- [Strimzi documentation](https://strimzi.io/docs/operators/latest/full/overview)
