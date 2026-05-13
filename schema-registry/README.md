# Schema Registry

This directory contains raw Kubernetes manifests for deploying Confluent Schema Registry alongside the Kafka cluster described in [`../kafka/README.md`](../kafka/README.md).

Return to the [root README](../README.md) for the overall repository layout and deployment order.

## Included manifests

- `schema-registry.yaml`: deployment and runtime configuration.
- `service.yaml`: in-cluster service on port `8081`.
- `ingress.yaml`: optional external access with TLS and basic authentication.

## Prerequisites

- A reachable Kafka cluster.
- A SCRAM Secret named `kafka-kraft-user` with the `sasl.jaas.config` key expected by `schema-registry.yaml`.
- The `kafka` namespace.
- An ingress controller and cert-manager if you want external access.

## Required customizations

Update these values before applying the manifests:

- Kafka bootstrap address in `schema-registry.yaml`.
- Ingress host in `ingress.yaml` (`schema-registry.example.com` by default).
- The ingress auth secret name `schema-registry-basic-auth`.
- Topic replication settings if your Kafka cluster has fewer than three brokers.

Create the ingress auth secret if you keep ingress enabled:

```sh
htpasswd -nbB admin change-me > auth
kubectl -n kafka create secret generic schema-registry-basic-auth --from-file=auth
rm auth
```

## Deployment

```sh
kubectl apply -f service.yaml
kubectl apply -f schema-registry.yaml
kubectl apply -f ingress.yaml
```

## Validation

```sh
kubectl get deploy,svc,ingress -n kafka
kubectl logs deployment/schema-registry -n kafka
```

Register a schema after replacing the host and credentials:

```sh
curl -u admin:change-me -X POST https://schema-registry.example.com/subjects/test-value/versions \
  -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  -d '{
    "schema": "{\"type\": \"record\", \"name\": \"Test\", \"fields\": [{\"name\": \"field1\", \"type\": \"string\"}]}"
  }'
```

Read the latest version:

```sh
curl -u admin:change-me https://schema-registry.example.com/subjects/test-value/versions/latest
```

## Security notes

- The manifests use `SASL_PLAINTEXT` when connecting to Kafka. Switch to TLS or `SASL_SSL` if your environment requires encrypted broker traffic.
- Do not commit the ingress basic-auth secret or the Kafka SCRAM credentials.

## Uninstall

```sh
kubectl delete -f ingress.yaml
kubectl delete -f schema-registry.yaml
kubectl delete -f service.yaml
```
