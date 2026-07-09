# Helm Basics - Complete Notes

# What is Helm?

Helm is the package manager for Kubernetes.

Just like:
- apt → Ubuntu packages
- yum → CentOS packages
- npm → Node.js packages

Helm manages Kubernetes applications using **Charts**.

A Helm chart is a collection of Kubernetes YAML files bundled together.

---

# Why Helm?

Without Helm

Create every resource manually.

- Deployment.yaml
- Service.yaml
- ConfigMap.yaml
- Secret.yaml
- Ingress.yaml
- HPA.yaml

Run

```bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl apply -f ingress.yaml
kubectl apply -f configmap.yaml
```

This becomes difficult for large applications.

---

With Helm

Everything is packaged into one Chart.

Just run

```bash
helm install my-app chart-name
```

Helm creates all Kubernetes resources automatically.

---

# Helm Architecture

```
Developer
      |
      |
   Helm Chart
      |
      |
helm install
      |
      |
 Kubernetes API Server
      |
      |
Deployment
Service
Ingress
ConfigMap
Secret
HPA
PVC
```

---

# Helm Installation

Download Helm

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-4
```

Give execute permission

```bash
chmod 700 get_helm.sh
```

Install Helm

```bash
./get_helm.sh
```

Verify installation

```bash
helm version
```

Example Output

```bash
version.BuildInfo{
Version:"v4.x.x"
}
```

---

# Create a Helm Chart

Command

```bash
helm create node-js-app
```

Output

```
node-js-app/
├── Chart.yaml
├── values.yaml
├── charts/
└── templates/
```

---

# Folder Explanation

## Chart.yaml

Contains chart information.

Example

```yaml
apiVersion: v2
name: node-js-app
description: Node.js Helm Chart
type: application
version: 0.1.0
appVersion: "1.0"
```

Meaning

| Field | Description |
|--------|-------------|
| apiVersion | Helm chart API version |
| name | Chart name |
| description | Description |
| type | application/library |
| version | Chart version |
| appVersion | Application version |

---

## values.yaml

Stores configurable values.

Example

```yaml
replicaCount: 2

image:
  repository: nginx
  tag: latest
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80
```

Instead of editing Deployment YAML directly, Helm reads these values.

---

## templates/

Contains Kubernetes manifests.

Example

```
deployment.yaml
service.yaml
serviceaccount.yaml
ingress.yaml
hpa.yaml
_helpers.tpl
NOTES.txt
```

---

## charts/

Stores dependency charts.

Usually empty unless dependencies are added.

---

# Changing Image

Default

```yaml
image:
  repository: nginx
```

Example

```yaml
image:
  repository: kuldeepdevda/node-app
  tag: latest
```

---

# Changing Replica Count

Default

```yaml
replicaCount: 1
```

Example

```yaml
replicaCount: 3
```

Helm automatically updates Deployment replicas.

---

# Check Chart Structure

```bash
ls
```

Example

```
Chart.yaml
charts
templates
values.yaml
```

---

# Package the Chart

Move outside the chart directory.

```
helm/
├── node-js-app/
```

Run

```bash
helm package node-js-app/
```

Output

```
Successfully packaged chart and saved it to:

node-js-app-0.1.0.tgz
```

Now directory becomes

```
helm/
├── node-js-app/
├── node-js-app-0.1.0.tgz
```

The `.tgz` file is the packaged Helm chart.

---

# Install the Chart

Using chart directory

```bash
helm install dev-node-app node-js-app \
-n node-js-app \
--create-namespace
```

Meaning

| Part | Description |
|------|-------------|
| helm install | Install chart |
| dev-node-app | Release name |
| node-js-app | Chart name/path |
| -n node-js-app | Namespace |
| --create-namespace | Create namespace if not present |

---

# Verify Installation

List Helm releases

```bash
helm list -A
```

Check pods

```bash
kubectl get pods -n node-js-app
```

Check deployment

```bash
kubectl get deployment -n node-js-app
```

Check service

```bash
kubectl get svc -n node-js-app
```

---

# Upgrade a Chart

Suppose replicaCount changed from

```yaml
1
```

to

```yaml
5
```

Run

```bash
helm upgrade dev-node-app node-js-app -n node-js-app
```

Helm updates only the changed resources.

---

# Rollback

View revisions

```bash
helm history dev-node-app -n node-js-app
```

Rollback

```bash
helm rollback dev-node-app 1 -n node-js-app
```

Rollback to revision 1.

---

# Uninstall

```bash
helm uninstall dev-node-app -n node-js-app
```

This removes all Kubernetes resources created by the release.

---

# Useful Commands

Check version

```bash
helm version
```

Create chart

```bash
helm create node-js-app
```

Package chart

```bash
helm package node-js-app/
```

Install chart

```bash
helm install dev-node-app node-js-app -n node-js-app --create-namespace
```

List releases

```bash
helm list -A
```

Upgrade

```bash
helm upgrade dev-node-app node-js-app -n node-js-app
```

History

```bash
helm history dev-node-app -n node-js-app
```

Rollback

```bash
helm rollback dev-node-app 1 -n node-js-app
```

Delete release

```bash
helm uninstall dev-node-app -n node-js-app
```

---

# Real-Time Example

## Step 1

Install Helm

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-4
chmod 700 get_helm.sh
./get_helm.sh
```

---

## Step 2

Create chart

```bash
helm create node-js-app
```

---

## Step 3

Open

```
values.yaml
```

Update

```yaml
replicaCount: 3

image:
  repository: kuldeepdevda/node-app
  tag: latest
```

---

## Step 4

Package chart

```bash
helm package node-js-app/
```

Output

```
node-js-app-0.1.0.tgz
```

---

## Step 5

Install chart

```bash
helm install dev-node-app node-js-app \
-n node-js-app \
--create-namespace
```

---

## Step 6

Verify

```bash
kubectl get all -n node-js-app
```

Example Output

```
NAME                                    READY   STATUS
pod/dev-node-app-xxxxx                  1/1     Running

NAME                     TYPE        CLUSTER-IP
service/dev-node-app     ClusterIP

NAME                               READY
deployment.apps/dev-node-app       3/3

NAME                               DESIRED
replicaset.apps/dev-node-app       3
```

---

# Helm Interview Questions

### What is Helm?

Helm is the package manager for Kubernetes that simplifies application deployment using Charts.

---

### What is a Helm Chart?

A Helm Chart is a package containing Kubernetes YAML templates and configuration files.

---

### What is values.yaml?

`values.yaml` stores configurable values such as image name, replica count, ports, and service type.

---

### What is Chart.yaml?

It contains chart metadata like name, version, description, and appVersion.

---

### Difference between `helm install` and `kubectl apply`?

| Helm | kubectl |
|-------|----------|
| Uses Charts | Uses individual YAML files |
| Supports versioning | No built-in versioning |
| Supports rollback | Manual rollback |
| Supports upgrades | Manual updates |

---
# Helm Release Name

A **Release Name** is the unique name given to an installed Helm Chart in a Kubernetes cluster.

In most organizations, the release name represents the **environment**.

Example:

| Environment | Release Name |
|-------------|--------------|
| Development | dev-node-app |
| QA | qa-node-app |
| UAT | uat-node-app |
| Production | prod-node-app |

This allows the same Helm Chart to be deployed multiple times for different environments.

Example Commands

Development

```bash
helm install dev-node-app node-js-app \
-n dev-node-app \
--create-namespace
```

QA

```bash
helm install qa-node-app node-js-app \
-n qa-node-app \
--create-namespace
```

UAT

```bash
helm install uat-node-app node-js-app \
-n uat-node-app \
--create-namespace
```

Production

```bash
helm install prod-node-app node-js-app \
-n prod-node-app \
--create-namespace
```

Here,

- **dev-node-app** → Release Name (Development Environment)
- **qa-node-app** → Release Name (QA Environment)
- **uat-node-app** → Release Name (UAT Environment)
- **prod-node-app** → Release Name (Production Environment)

The same chart (`node-js-app`) is reused for every environment; only the **release name** and usually the **namespace** change.

---

# Example

Chart Name

```
node-js-app
```

Deploy the same chart into different environments:

```bash
helm install dev-node-app node-js-app -n dev-node-app --create-namespace

helm install qa-node-app node-js-app -n qa-node-app --create-namespace

helm install uat-node-app node-js-app -n uat-node-app --create-namespace

helm install prod-node-app node-js-app -n prod-node-app --create-namespace
```

This creates four separate Helm releases, one for each environment, while using the same Helm Chart.

---

# Summary

- Helm is the package manager for Kubernetes.
- A Chart is a package of Kubernetes manifests.
- `Chart.yaml` stores metadata.
- `values.yaml` stores configurable values.
- `templates/` contains Kubernetes YAML templates.
- `helm package` creates a `.tgz` archive.
- `helm install` deploys the application.
- `helm upgrade` updates a release.
- `helm rollback` restores a previous revision.
- `helm uninstall` removes the release.
