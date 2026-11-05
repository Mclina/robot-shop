# Robot Shop — Observability Stack

## Overview

This directory contains the observability configuration for the Robot Shop Kubernetes deployment.

The stack includes:

- Prometheus — metrics collection and monitoring.
- Node Exporter — host-level metrics.
- kube-state-metrics — Kubernetes resource metrics.
- Grafana — metrics visualisation and dashboards.
- Fluent Bit — container log collection.
- OpenTelemetry Collector — OTLP telemetry reception and processing.

## Directory structure

```text
observability/
├── prometheus/
│   ├── prometheus.yml
│   └── prometheus.yaml
├── grafana/
│   ├── dashboards/
│   │   └── robot-shop.json
│   └── provisioning/
│       ├── dashboards/
│       │   └── dashboards.yaml
│       └── datasources/
│           └── prometheus.yaml
├── fluent-bit/
│   └── values.yaml
└── opentelemetry/
    └── collector.yaml
```

## Prerequisites

- Kubernetes cluster
- kubectl
- Helm
- Robot Shop deployed in the `robot-shop` namespace
- An `observability` namespace

Create the namespace:

```bash
kubectl create namespace observability
```

## 1. Prometheus

Deploy Prometheus:

```bash
kubectl apply -f observability/prometheus/prometheus.yaml
```

## 2. Node Exporter

Add the Prometheus Community Helm repository:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

Install Node Exporter:

```bash
helm upgrade --install node-exporter prometheus-community/prometheus-node-exporter \
  -n observability \
  --set resources.requests.cpu=10m \
  --set resources.requests.memory=32Mi \
  --set resources.limits.cpu=100m \
  --set resources.limits.memory=128Mi
```

## 3. kube-state-metrics

```bash
helm upgrade --install kube-state-metrics prometheus-community/kube-state-metrics \
  -n observability \
  --set resources.requests.cpu=10m \
  --set resources.requests.memory=32Mi \
  --set resources.limits.cpu=100m \
  --set resources.limits.memory=128Mi
```

## 4. Grafana

Add the Grafana Helm repository:

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

Install Grafana:

```bash
helm upgrade --install grafana grafana/grafana \
  -n observability \
  --set persistence.enabled=false \
  --set resources.requests.cpu=50m \
  --set resources.requests.memory=128Mi \
  --set resources.limits.cpu=200m \
  --set resources.limits.memory=256Mi
```

The datasource and dashboard provisioning files are stored in the Grafana subdirectories. They must be mounted into the Grafana deployment for automatic provisioning.

Dashboard: `Robot Shop Kubernetes Monitoring`.

## 5. Fluent Bit

Add the Helm repository:

```bash
helm repo add fluent https://fluent.github.io/helm-charts
helm repo update
```

Install Fluent Bit:

```bash
helm upgrade --install fluent-bit fluent/fluent-bit \
  -n observability \
  -f observability/fluent-bit/values.yaml \
  --set resources.requests.cpu=10m \
  --set resources.requests.memory=32Mi \
  --set resources.limits.cpu=100m \
  --set resources.limits.memory=128Mi
```

The current output is stdout for demonstration and verification. It is not persistent centralised log storage.

## 6. OpenTelemetry Collector

Deploy the Collector:

```bash
kubectl apply -f observability/opentelemetry/collector.yaml
```

OTLP endpoints:

- gRPC: port 4317
- HTTP: port 4318

The current Collector uses the debug exporter. Applications must send OTLP telemetry to it before traces, metrics or logs can be observed.

## 7. Verification

Check observability pods:

```bash
kubectl get pods -n observability
```

Check services:

```bash
kubectl get services -n observability
```

Check Prometheus targets:

```bash
kubectl port-forward -n observability svc/prometheus 9090:9090
```

Open `http://localhost:9090/targets`.

Check Fluent Bit logs:

```bash
kubectl logs -n observability daemonset/fluent-bit --tail=30
```

Check OpenTelemetry Collector logs:

```bash
kubectl logs -n observability deployment/otel-collector --tail=30
```

## 8. Known limitations

- Prometheus uses ephemeral storage in the playground configuration.
- Grafana persistence is disabled in the playground configuration.
- Fluent Bit currently forwards logs to stdout rather than a persistent log backend.
- OpenTelemetry Collector is deployed, but application instrumentation and end-to-end telemetry verification remain outstanding.
- Playground resources and storage may be lost when the temporary environment expires.

## Project status

The observability components were deployed and their workloads verified as running in the free Kubernetes playground. Further work is required to demonstrate end-to-end application metrics, centralised logging and OpenTelemetry telemetry ingestion.
