# Monitoring Kubernetes with Helm (Prometheus + Grafana)

## Overview

Helm is the package manager for Kubernetes. Instead of manually creating
dozens of YAML files, Helm installs a complete application using a
**chart**.

In this setup, we use the **kube-prometheus-stack** Helm chart, which
installs:

-   Prometheus (metrics collection)
-   Grafana (visualization)
-   Alertmanager
-   Node Exporter
-   kube-state-metrics

  
**cmd**

Repository: helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

------------------------------------------------------------------------

# Step 1: Install Helm

``` bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-4
chmod 700 get_helm.sh
./get_helm.sh
```

### What happens?

-   Downloads the Helm installation script.
-   Makes it executable.
-   Installs Helm on your machine.

Verify:

``` bash
helm version
```

------------------------------------------------------------------------

# Step 2: Install the Prometheus Community Repository

``` bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

### Why?

Helm needs to know where the chart is stored.

------------------------------------------------------------------------

# Step 3: Install kube-prometheus-stack

``` bash
helm install prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitor \
  --create-namespace \
  --set prometheus.service.type=NodePort \
  --set prometheus.service.nodePort=30000 \
  --set grafana.service.type=NodePort \
  --set grafana.service.nodePort=31000
```

### Explanation

-   `helm install` → Installs a Helm chart.
-   `prometheus-stack` → Release name.
-   `prometheus-community/kube-prometheus-stack` → Chart name.
-   `--namespace monitor` → Install into the `monitor` namespace.
-   `--create-namespace` → Creates the namespace if it does not exist.
-   `prometheus.service.type=NodePort` → Exposes Prometheus as a
    NodePort service.
-   `prometheus.service.nodePort=30000` → Prometheus NodePort.
-   `grafana.service.type=NodePort` → Exposes Grafana as a NodePort
    service.
-   `grafana.service.nodePort=31000` → Grafana NodePort.

------------------------------------------------------------------------

# Step 4: Verify Installation

``` bash
kubectl get pods -n monitor
kubectl get svc -n monitor
```

Expected components:

-   Prometheus
-   Grafana
-   Alertmanager
-   Node Exporter
-   kube-state-metrics

------------------------------------------------------------------------

# Step 5: Access Prometheus

``` bash
kubectl port-forward svc/prometheus-stack-kube-prom-prometheus \
-n monitor \
9090:9090 \
--address=0.0.0.0 &
```

Open:

    http://localhost:9090

### Why port-forward?

It forwards local port **9090** to the Prometheus service running inside
Kubernetes.

------------------------------------------------------------------------

# Step 6: Access Grafana

``` bash
kubectl port-forward svc/prometheus-stack-grafana \
-n monitor \
3000:80 \
--address=0.0.0.0 &
```

Open:

    http://localhost:3000

Local port **3000** forwards to Grafana service port **80**.

------------------------------------------------------------------------

# Step 7: Get Grafana Password

``` bash
kubectl get secret prometheus-stack-grafana \
-n monitor \
-o jsonpath="{.data.admin-password}" | base64 --decode
```

Default username:

    admin

The command decodes the stored Kubernetes Secret to display the
password.

------------------------------------------------------------------------

# Architecture

``` text
Kubernetes Cluster
        │
        ▼
 Helm installs kube-prometheus-stack
        │
 ├── Prometheus
 ├── Grafana
 ├── Alertmanager
 ├── Node Exporter
 └── kube-state-metrics

Prometheus ---> Collects Metrics
Grafana -----> Displays Dashboards
```

# Summary

1.  Install Helm.
2.  Add the Prometheus Helm repository.
3.  Install `kube-prometheus-stack`.
4.  Verify pods and services.
5.  Port-forward Prometheus (9090).
6.  Port-forward Grafana (3000).
7.  Retrieve the Grafana admin password from the Kubernetes Secret.
