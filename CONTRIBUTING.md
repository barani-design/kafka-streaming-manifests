# Contributing

## Scope

This repository contains Kubernetes manifests and documentation for a Kafka-centered platform deployment. Keep changes focused on manifests, examples, validation, and accompanying docs.

## Workflow

- Open an issue or discussion before making large behavior or structure changes.
- Keep component-specific details in the relevant subdirectory README.
- Keep cross-component assumptions documented in the root `README.md`.

## Pull Request Checklist

- Do not commit secrets, passwords, internal hostnames, internal IPs, or private registry references.
- Update the relevant component README when deployment assumptions change.
- Keep placeholder values clearly marked as placeholders.
- Validate changed YAML files before submitting.
