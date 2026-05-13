# ksqlDB

This directory contains raw Kubernetes manifests for deploying a ksqlDB server, a helper CLI pod, a persistent volume claim, and an optional ingress.

Return to the [root README](../README.md) for the overall repository layout and deployment order.

## Included manifests

- `ksqldb.yaml`: ksqlDB server deployment.
- `service.yaml`: in-cluster service on port `8088`.
- `ksqldb-cli.yaml`: helper deployment for interactive `ksql` sessions.
- `pvc.yaml`: persistent state volume claim.
- `ingress.yaml`: optional external access with TLS and basic authentication.

## Prerequisites

- A reachable Kafka cluster and a SCRAM Secret named `kafka-kraft-user`.
- A reachable Schema Registry endpoint at the URL configured in `ksqldb.yaml`.
- The `kafka` namespace.
- A storage provisioner for `pvc.yaml`.
- An ingress controller and cert-manager if you want external access.

## Required customizations

Review these values before deployment:

- Kafka bootstrap address in `ksqldb.yaml`.
- Schema Registry URL in `ksqldb.yaml`.
- Persistent volume sizing in `pvc.yaml`.
- Ingress host and secret name in `ingress.yaml`.
- ksqlDB internal topic replication settings if your Kafka cluster has fewer than three brokers.

Create the ingress auth secret if you keep ingress enabled:

```sh
htpasswd -nbB admin change-me > auth
kubectl -n kafka create secret generic ksqldb-basic-auth --from-file=auth
rm auth
```

## Deployment

```sh
kubectl apply -f pvc.yaml
kubectl apply -f service.yaml
kubectl apply -f ksqldb.yaml
kubectl apply -f ksqldb-cli.yaml
kubectl apply -f ingress.yaml
```

## Validation

```sh
kubectl get deploy,pvc,svc,ingress -n kafka
kubectl logs deployment/ksqldb -n kafka
kubectl exec -it deployment/ksqldb-cli -n kafka -- ksql http://ksqldb.kafka.svc:8088
```

Example stream definition:

```sql
CREATE STREAM example_events (
  event_id STRING KEY,
  payload STRING
) WITH (
  KAFKA_TOPIC = 'external.lorawan.tti.region-a.raw.v1',
  VALUE_FORMAT = 'JSON'
);
```

## Security notes

- The server connects to Kafka with `SASL_PLAINTEXT`. Switch to TLS or `SASL_SSL` if your environment requires encrypted broker traffic.
- Do not commit basic-auth secrets or Kafka SCRAM credentials.

## Uninstall

```sh
kubectl delete -f ingress.yaml
kubectl delete -f ksqldb-cli.yaml
kubectl delete -f ksqldb.yaml
kubectl delete -f service.yaml
kubectl delete -f pvc.yaml
```

## References

- [ksqlDB documentation](https://docs.ksqldb.io/en/latest/)
- [Strimzi Kafka Operator issue 3294](https://github.com/strimzi/strimzi-kafka-operator/issues/3294)