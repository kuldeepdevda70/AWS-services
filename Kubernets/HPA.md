# Kubernetes Horizontal Pod Autoscaler (HPA)

# What is Horizontal Pod Autoscaler (HPA)?

**Horizontal Pod Autoscaler (HPA)** is a Kubernetes resource that **automatically increases or decreases the number of Pods** in a Deployment, ReplicaSet, or StatefulSet based on resource usage such as **CPU** or **Memory**.

Instead of manually changing replicas, HPA continuously monitors the application and adjusts the number of Pods according to the configured metrics.

---

# Why Do We Need HPA?

Suppose your application has only **1 Pod**.

- During normal traffic, one Pod is enough.
- Suddenly, many users access the application.
- CPU usage becomes very high.
- One Pod cannot handle all requests.

Without HPA:
- Application becomes slow.
- Requests may fail.
- Users get poor performance.

With HPA:
- Kubernetes automatically creates more Pods.
- Traffic is distributed among all Pods.
- When traffic decreases, extra Pods are removed automatically.

---

# Real-Life Example

Imagine a restaurant.

- Normally, **1 chef** is enough.
- During lunch time, many customers arrive.
- Restaurant hires **4 more chefs** temporarily.
- After rush hour, extra chefs leave.

HPA works exactly the same way.

Pods = Chefs

Customers = Incoming Requests

---

# How HPA Works

```
Users
   │
   ▼
Deployment
   │
   ▼
Current Pods = 1

      │
      ▼
Metrics Server checks CPU Usage

      │
      ▼
CPU > Target (5%)

      │
      ▼
HPA increases replicas

1 Pod
 ↓
2 Pods
 ↓
3 Pods
 ↓
5 Pods (Maximum)

When CPU becomes low

5 Pods
 ↓
4 Pods
 ↓
3 Pods
 ↓
1 Pod (Minimum)
```

---

# Prerequisites

Before using HPA, you must have:

## 1. Kubernetes Cluster

Running Kubernetes cluster.

Example:

```bash
kubectl get nodes
```

---

## 2. Metrics Server

- If you are using a Kind cluster install Metrics Server
```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```
- Edit the Metrics Server Deployment
```bash
kubectl -n kube-system edit deployment metrics-server
```
- Add the security bypass to deployment under `container.args`
```bash
- --kubelet-insecure-tls
- --kubelet-preferred-address-types=InternalIP,Hostname,ExternalIP
```
- Restart the deployment
```bash
kubectl -n kube-system rollout restart deployment metrics-server
```
- Verify if the metrics server is running
```bash
kubectl get pods -n kube-system
kubectl top nodes
```


HPA requires the **Metrics Server**.

Without Metrics Server, HPA cannot monitor CPU or Memory.

Verify:

```bash
kubectl top nodes
```

or

```bash
kubectl top pods
```

If these commands work, Metrics Server is installed.

---

## 3. Resource Requests and Limits

HPA calculates CPU utilization based on **requests**.

Without resource requests, HPA will not work properly.

Example:

```yaml
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"

  limits:
    cpu: "200m"
    memory: "256Mi"
```

Meaning:

| Resource | Request | Limit |
|----------|---------|--------|
| CPU | 100m | 200m |
| Memory | 128Mi | 256Mi |

---

# Project Structure

```
k8s/

├── node-namespace.yml
├── deployment.yml
├── service.yml
└── hpa.yml
```

---

# Step 1 – Create Namespace

**node-namespace.yml**

```yaml
apiVersion: v1
kind: Namespace

metadata:
  name: node-app
```

Apply:

```bash
kubectl apply -f node-namespace.yml
```

Verify:

```bash
kubectl get ns
```

---

# Step 2 – Create Deployment

**deployment.yml**

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: node-deployment
  namespace: nginx

spec:
  replicas: 1

  selector:
    matchLabels:
      app: node-app

  template:
    metadata:
      labels:
        app: node-app

    spec:
      containers:
      - name: node-app
        image: kuldeep9165/k8sdjengo

        ports:
        - containerPort: 8000

        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"

          limits:
            cpu: "200m"
            memory: "256Mi"
```

Apply:

```bash
kubectl apply -f deployment.yml
```

Verify:

```bash
kubectl get deployment -n nginx
```

---

# Step 3 – Create Service

**service.yml**

```yaml
apiVersion: v1
kind: Service

metadata:
  name: app-node
  namespace: nginx

spec:
  selector:
    app: node-app

  ports:
  - port: 8000
    targetPort: 8000

  type: ClusterIP
```

Apply:

```bash
kubectl apply -f service.yml
```

Verify:

```bash
kubectl get svc -n nginx
```

---

# Step 4 – Create HPA

**hpa.yml**

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler

metadata:
  name: node-hpa
  namespace: nginx

spec:

  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: node-deployment

  minReplicas: 1

  maxReplicas: 5

  metrics:
  - type: Resource

    resource:
      name: cpu

      target:
        type: Utilization
        averageUtilization: 5
```

Apply:

```bash
kubectl apply -f hpa.yml
```

Verify:

```bash
kubectl get hpa -n nginx
```

---

# Understanding Every Field

## apiVersion

```yaml
apiVersion: autoscaling/v2
```

Uses the latest HPA API.

---

## kind

```yaml
kind: HorizontalPodAutoscaler
```

Creates an HPA resource.

---

## scaleTargetRef

```yaml
scaleTargetRef:
  apiVersion: apps/v1
  kind: Deployment
  name: node-deployment
```

Tells HPA which Deployment should be scaled.

---

## minReplicas

```yaml
minReplicas: 1
```

Minimum number of Pods.

HPA will never go below 1 Pod.

---

## maxReplicas

```yaml
maxReplicas: 5
```

Maximum Pods allowed.

Even if CPU reaches 100%, HPA will not create more than 5 Pods.

---

## Metrics

```yaml
metrics:
- type: Resource
```

Monitor Kubernetes resources.

Possible resources:

- CPU
- Memory

---

## CPU

```yaml
name: cpu
```

Monitor CPU usage.

---

## Target Type

```yaml
type: Utilization
```

Compare CPU usage percentage against the requested CPU.

---

## averageUtilization

```yaml
averageUtilization: 5
```

Target CPU utilization is **5%**.

If average CPU usage goes above 5%, HPA starts adding Pods.

> **Note:** A target of **5%** is very low and is mainly useful for testing. In production, values like **50–80%** are more common.

---

# Apply Everything

```bash
kubectl apply -f node-namespace.yml
kubectl apply -f deployment.yml
kubectl apply -f service.yml
kubectl apply -f hpa.yml
```

---

# Verify Resources

Deployment

```bash
kubectl get deployment -n nginx
```

Pods

```bash
kubectl get pods -n nginx
```

Service

```bash
kubectl get svc -n nginx
```

HPA

```bash
kubectl get hpa -n nginx
```

Detailed HPA information

```bash
kubectl describe hpa node-hpa -n nginx
```

---

# Monitor HPA

Watch HPA continuously:

```bash
kubectl get hpa -n nginx -w
```

Watch Pods:

```bash
kubectl get pods -n nginx -w
```

---

# Generate CPU Load (Testing)

Open a temporary Pod:

```bash
kubectl run load-generator \
  --rm -it \
  --image=busybox \
  -- /bin/sh
```

Inside the Pod:

```sh
while true; do
  wget -q -O- http://app-node:8000
done
```

Observe HPA in another terminal:

```bash
kubectl get hpa -n nginx -w
```

Observe Pods:

```bash
kubectl get pods -n nginx -w
```

You should see the replica count increase if CPU usage exceeds the configured target.

---

# Useful Commands

View Pods

```bash
kubectl get pods -n nginx
```

View Deployment

```bash
kubectl get deployment -n nginx
```

View HPA

```bash
kubectl get hpa -n nginx
```

Describe HPA

```bash
kubectl describe hpa node-hpa -n nginx
```

CPU Usage

```bash
kubectl top pod -n nginx
```

Delete HPA

```bash
kubectl delete hpa node-hpa -n nginx
```

---

# Interview Questions

### 1. What is HPA?

Horizontal Pod Autoscaler automatically scales the number of Pods based on CPU, Memory, or other metrics.

---

### 2. What does HPA scale?

It scales Pods horizontally by increasing or decreasing the number of replicas.

---

### 3. What is required for HPA?

- Metrics Server
- Deployment/StatefulSet/ReplicaSet
- Resource Requests (CPU/Memory)

---

### 4. Does HPA create new nodes?

No.

HPA only creates or removes Pods.

For node scaling, Kubernetes uses the **Cluster Autoscaler**.

---

### 5. Why are CPU Requests important?

HPA calculates utilization based on the requested CPU.

Without CPU requests, HPA cannot accurately determine utilization.

---

### 6. Difference between HPA and VPA?

| HPA | VPA |
|------|-----|
| Scales Pods | Changes CPU/Memory of existing Pods |
| Horizontal Scaling | Vertical Scaling |
| Adds/Removes Pods | Increases/Decreases resources per Pod |

---

### 7. What happens if CPU exceeds the target?

HPA automatically increases the number of Pods until it reaches `maxReplicas`.

---

### 8. What happens when traffic decreases?

HPA automatically reduces the number of Pods, but never below `minReplicas`.

---

# Important Note About Your YAML

Your **namespace file** creates:

```yaml
name: node-app
```

But your **Deployment, Service, and HPA** use:

```yaml
namespace: nginx
```

This is inconsistent.

Choose **one namespace** and use it everywhere.

For example, if you want to use the `node-app` namespace, update all files:

```yaml
namespace: node-app
```

Or, if you intend to use `nginx`, create the `nginx` namespace instead.

Keeping the namespace consistent is essential for all resources to work together.

---

# Summary

- HPA automatically scales Pods based on metrics.
- Metrics Server is mandatory.
- CPU requests are required for CPU-based scaling.
- `minReplicas` defines the minimum number of Pods.
- `maxReplicas` defines the maximum number of Pods.
- `averageUtilization` is the target CPU usage percentage.
- HPA performs **horizontal scaling** (Pods), not node scaling.
- Use `kubectl get hpa` and `kubectl describe hpa` to monitor scaling behavior.
