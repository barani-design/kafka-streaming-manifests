# Camel Connect

This directory contains raw Kubernetes manifests for a Strimzi `KafkaConnect` cluster that uses the Camel Netty HTTP connector to receive webhook requests and publish them to Kafka topics.

Return to the [root README](../README.md) for the overall repository layout and deployment order.

## Included manifests

- `connect.yaml`: `KafkaConnect` cluster with the Camel Netty HTTP connector plugin.
- `connector.yaml`: example webhook connector resources.
- `topic.yaml`: example Kafka topics for external ingress data and internal processing stages.
- `service.yaml`: one service per webhook port.
- `ingress.yaml`: optional external ingress rules for the example webhook set.
- `network_policy.yaml`: ingress rules for the webhook ports on the Kafka Connect pods.
- `webhook-logger-deployment.yaml`, `webhook-logger-service.yaml`, `webhook-logger-ingress.yaml`: optional helper deployment for debugging webhook traffic.
- `images/camel-connect-diagram.puml`: canonical PlantUML source for the example topology.

## Prerequisites

- Strimzi Kafka Operator installed with connector resources enabled.
- A reachable Kafka cluster and a SCRAM Secret named `kafka-kraft-user`.
- The `kafka` namespace.
- A container registry path you can push the built Kafka Connect image to.
- An ingress controller and cert-manager if you want external webhook access.

## Required customizations

Review these values before deployment:

- Output image in `connect.yaml` (`ghcr.io/example/camel-connect-cluster:latest` is a placeholder).
- Webhook paths, ports, and topic names in `connector.yaml`.
- Ingress hosts in `ingress.yaml`.
- Node labels, CPU, and memory values in `connect.yaml`.
- Network policy scope in `network_policy.yaml`.

Create the ingress auth secret if you keep the authenticated ingress rules:

```sh
htpasswd -nbB webhook change-me > auth
kubectl -n kafka create secret generic webhook-basic-auth --from-file=auth
rm auth
```

## Example webhook mapping

- `https://camel-connect-webhook.example.com/webhook-lorawan-tti-region-a` -> `external.lorawan.tti.region-a.raw.v1`
- `https://camel-connect-webhook.example.com/webhook-lorawan-tti-region-b` -> `external.lorawan.tti.region-b.raw.v1`
- `https://camel-connect-webhook.example.com/webhook-lorawan-tti-region-c` -> `external.lorawan.tti.region-c.raw.v1`
- `https://camel-connect-webhook.example.com/webhook-vendor-a` -> `external.iot.vendor-a.raw.v1`
- `https://camel-connect-webhook-vendor-b.example.com/webhook-vendor-b-events` -> `external.iot.vendor-b.events.raw.v1`
- `https://camel-connect-webhook-vendor-b.example.com/webhook-vendor-b-telemetry` -> `external.iot.vendor-b.telemetry.raw.v1`

## Deployment order

```sh
kubectl apply -f topic.yaml
kubectl apply -f connect.yaml
kubectl apply -f connector.yaml
kubectl apply -f service.yaml
kubectl apply -f ingress.yaml
kubectl apply -f network_policy.yaml
```

Apply the optional webhook logger only when you need a debugging endpoint:

```sh
kubectl apply -f webhook-logger-deployment.yaml
kubectl apply -f webhook-logger-service.yaml
kubectl apply -f webhook-logger-ingress.yaml
```

## Validation

```sh
kubectl get kafkaconnect,kafkaconnector,svc,ingress,networkpolicy -n kafka
kubectl logs deployment/camel-connect-cluster-connect -n kafka
```

In-cluster test:

```sh
curl -X POST http://camel-connect-webhook-vendor-a-service.kafka.svc:61003/webhook-vendor-a \
  -H "Content-Type: application/json" \
  -d '{"hello":"from cluster"}'
```

External test after updating DNS and credentials:

```sh
curl -X POST https://camel-connect-webhook.example.com/webhook-lorawan-tti-region-a \
  -u webhook:change-me \
  -H "Content-Type: application/json" \
  -d '{"hello":"from internet"}'
```

## Security notes

- The repository no longer ships a committed basic-auth Secret. Create `webhook-basic-auth` separately before enabling protected ingress routes.
- `network_policy.yaml` currently allows traffic from any namespace to the webhook ports. Tighten it before production use.
- The connector receives plain HTTP inside the cluster. If you need end-to-end encryption, terminate TLS closer to the workload or use a service mesh pattern.

## Uninstall

```sh
kubectl delete -f network_policy.yaml
kubectl delete -f ingress.yaml
kubectl delete -f service.yaml
kubectl delete -f connector.yaml
kubectl delete -f connect.yaml
kubectl delete -f topic.yaml
```

## References

- [Apache Camel Kafka Connector documentation](https://camel.apache.org/camel-kafka-connector/next/index.html)
- [Strimzi Kafka Connect builds](https://strimzi.io/docs/operators/latest/configuring.html#assembly-kafka-connect-build-str)

## Diagram

The topology diagram is maintained as PlantUML source in [`images/camel-connect-diagram.puml`](images/camel-connect-diagram.puml). A rendered image is optional; the `.puml` file is the authoritative source in this repository.