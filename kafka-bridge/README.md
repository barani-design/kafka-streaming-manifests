# Kafka Bridge

This directory contains raw Kubernetes manifests for deploying Strimzi Kafka Bridge and optionally exposing it through ingress.

Return to the [root README](../README.md) for the overall repository layout and deployment order.

## Included manifests

- `kafka-bridge.yaml`: `KafkaBridge` custom resource with SCRAM authentication to Kafka.
- `ingress.yaml`: optional external access with TLS and basic authentication.

## Prerequisites

- Strimzi Kafka Operator installed.
- A reachable Kafka cluster.
- A SCRAM Secret named `kafka-kraft-user`.
- The `kafka` namespace.
- An ingress controller and cert-manager if you want external access.

## Required customizations

Review these values before applying the manifests:

- Kafka bootstrap address in `kafka-bridge.yaml`.
- Ingress host in `ingress.yaml` (`kafka-bridge.example.com` by default).
- Basic-auth secret name `kafka-bridge-basic-auth`.

Create the ingress auth secret if you keep ingress enabled:

```sh
htpasswd -nbB admin change-me > auth
kubectl -n kafka create secret generic kafka-bridge-basic-auth --from-file=auth
rm auth
```

## Deployment

```sh
kubectl apply -f kafka-bridge.yaml
kubectl apply -f ingress.yaml
```

## Validation

```sh
kubectl get kafkabridge,ingress -n kafka
kubectl logs deployment/kafka-bridge-bridge -n kafka
```

List topics after replacing the host and credentials:

```sh
curl -u admin:change-me https://kafka-bridge.example.com/topics
```

## Security notes

- The bridge authenticates to Kafka with SCRAM credentials from `kafka-kraft-user`.
- The ingress manifest expects a separate basic-auth secret and should not be committed with real credentials.

## Uninstall

```sh
kubectl delete -f ingress.yaml
kubectl delete -f kafka-bridge.yaml
```

## Reference

- [Strimzi Kafka Bridge documentation](https://strimzi.io/docs/operators/latest/configuring.html#assembly-kafka-bridge-str)
