# Robot Shop — Observability Stack

## Overview

This directory contains the observability configuration for the Robot Shop Kubernetes deployment.

The stack includes:

- Prometheus — metrics collection and monitoring.
- Node Exporter — host-level metrics.
- Nginx Prometheus Exporter — Nginx web-server metrics.
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
│   ├── dashboard-configmap.yaml
│   ├── datasource-configmap.yaml
│   └── values.yaml
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

## 4. Nginx Exporter

The Robot Shop web deployment includes an Nginx Prometheus Exporter sidecar.

- Nginx exposes its status endpoint at `/nginx_status`.
- Access to the status endpoint is restricted to loopback.
- The exporter scrapes the endpoint through `127.0.0.1:8080`.
- The exporter exposes metrics on port `9113`.
- The internal `web-metrics` ClusterIP Service makes metrics available to Prometheus.

Prometheus scrape target:

`web-metrics.robot-shop.svc.cluster.local:9113`

The Grafana dashboard includes these Nginx panels:

- Active Nginx Connections — `nginx_connections_active`
- HTTP Requests per Second — `rate(nginx_http_requests_total[5m])`
- Connections Handled per Second — `rate(nginx_connections_handled[5m])`

These are web-server operational metrics, not Robot Shop business transaction metrics.


## 5. Grafana

Apply the Grafana dashboard and datasource ConfigMaps:

```bash
kubectl apply -f observability/grafana/dashboard-configmap.yaml
kubectl apply -f observability/grafana/datasource-configmap.yaml
```

Install Grafana using the Grafana Community Helm chart:

```bash
helm upgrade --install grafana \
  oci://ghcr.io/grafana-community/helm-charts/grafana \
  --version 13.5.0 \
  -n observability \
  -f observability/grafana/values.yaml
```

The Grafana sidecars automatically discover dashboard and datasource ConfigMaps labelled `grafana_dashboard: "1"` and `grafana_datasource: "1"` in the `observability` namespace.

Dashboard: `Robot Shop Kubernetes Monitoring`.

## 6. Fluent Bit

Install Fluent Bit using the official Fluent Helm chart:

```bash
helm upgrade --install fluent-bit \
  oci://ghcr.io/fluent/helm-charts/fluent-bit \
  --version 0.58.3 \
  -n observability \
  -f observability/fluent-bit/values.yaml
```

The current output is stdout for demonstration and verification. It is not persistent centralised log storage.

## 7. OpenTelemetry Collector

Deploy the Collector:

```bash
kubectl apply -f observability/opentelemetry/collector.yaml
```

OTLP endpoints:

- gRPC: port 4317
- HTTP: port 4318

The current Collector uses the debug exporter. Applications must send OTLP telemetry to it before traces, metrics or logs can be observed.

## 8. Verification

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

## 9. Known limitations

- Prometheus uses ephemeral storage in the playground configuration.
- Grafana persistence is disabled in the playground configuration.
- Fluent Bit currently forwards logs to stdout rather than a persistent log backend.
- OpenTelemetry Collector is deployed, but application instrumentation and end-to-end telemetry verification remain outstanding.
- Nginx Exporter and its Prometheus scrape configuration have passed local Helm/YAML validation; runtime scraping still needs verification in a Kubernetes cluster.
- Playground resources and storage may be lost when the temporary environment expires.

## Project status

The observability components were deployed and their workloads verified as running in the free Kubernetes playground. The Nginx Exporter sidecar, metrics Service, Prometheus scrape configuration and Grafana dashboard panels have passed local configuration validation. Runtime verification of these additions is outstanding. Further work is also required to demonstrate end-to-end application metrics, centralised logging and OpenTelemetry telemetry ingestion.
