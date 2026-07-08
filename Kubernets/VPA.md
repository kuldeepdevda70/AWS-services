# Kubernetes Vertical Pod Autoscaler (VPA)

## What is Vertical Pod Autoscaler (VPA)?

**Vertical Pod Autoscaler (VPA)** is a Kubernetes autoscaling component that automatically adjusts the **CPU** and **Memory** requests (and optionally limits) of a Pod based on its actual resource usage.

Unlike the Horizontal Pod Autoscaler (HPA), which increases or decreases the number of Pods, VPA increases or decreases the resources assigned to each individual Pod.

---

# Why Do We Need VPA?

Applications often start with manually configured CPU and Memory values.

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

Problems with manual configuration:

- Request is too high → Cluster resources are wasted.
- Request is too low → Application may become slow.
- Difficult to determine the correct values.
- Resource requirements change over time.

VPA solves this problem by continuously monitoring resource usage and recommending or automatically applying better values.

---

# How VPA Works

```
            +------------------------+
            |      Application       |
            +-----------+------------+
                        |
                 CPU / Memory Usage
                        |
                        v
          +---------------------------+
          |      Metrics Server       |
          +---------------------------+
                        |
                        v
          +---------------------------+
          |      VPA Recommender      |
          +---------------------------+
                        |
          Calculates Recommended
          CPU & Memory Requests
                        |
                        v
          +---------------------------+
          |       VPA Object          |
          +---------------------------+
                        |
          updateMode: Auto
                        |
                        v
          +---------------------------+
          |      VPA Updater          |
          +---------------------------+
                        |
        Deletes Pod (if required)
                        |
                        v
          +---------------------------+
          |   Deployment creates      |
          |   New Pod with updated    |
          |      resource requests    |
          +---------------------------+
```

---

# Components Required for VPA

## 1. Metrics Server

Purpose:

- Collects CPU usage
- Collects Memory usage
- Provides metrics to Kubernetes

Check:

```bash
kubectl top pods
kubectl top nodes
```

---

## 2. VPA Recommender

Purpose:

- Reads metrics from Metrics Server.
- Calculates the recommended CPU and Memory requests.
- Continuously updates recommendations.

Example recommendation:

```
CPU:
Target: 15m

Memory:
Target: 70Mi
```

---

## 3. VPA Updater

Purpose:

When `updateMode` is set to `Auto`, the updater:

- Checks if the running Pod's resources differ significantly from the recommendation.
- Evicts (deletes) the Pod if needed.
- The Deployment creates a new Pod.
- The new Pod starts with the recommended resource requests.

---

## 4. VPA Admission Controller

Purpose:

When a new Pod is created:

- Reads VPA recommendations.
- Updates the Pod's resource requests before it starts.

---

# VPA Update Modes

## 1. Off

No automatic changes.

Only recommendations are generated.

```yaml
updatePolicy:
  updateMode: Off
```

Use when you only want to monitor recommendations.

---

## 2. Initial

Recommendations are applied **only when the Pod is created**.

Existing Pods are not updated.

```yaml
updatePolicy:
  updateMode: Initial
```

---

## 3. Auto

Fully automatic mode.

```yaml
updatePolicy:
  updateMode: Auto
```

Behavior:

- Monitor usage
- Recommend values
- Evict Pod if necessary
- New Pod gets updated resource requests

---

## 4. Recreate

Works similarly to Auto but recreates Pods when applying changes.

(Some Kubernetes versions treat `Recreate` the same as `Auto`.)

---

# Your Deployment

## deployment.yml

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: apache-deployment
  namespace: apache

spec:
  replicas: 1

  selector:
    matchLabels:
      app: apache

  template:
    metadata:
      labels:
        app: apache

    spec:
      containers:
      - name: apache
        image: httpd:latest

        ports:
        - containerPort: 80

        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"

          limits:
            cpu: "200m"
            memory: "256Mi"
```

Current resources:

| Resource | Request | Limit |
|----------|---------|--------|
| CPU | 100m | 200m |
| Memory | 128Mi | 256Mi |

---

# Your VPA Configuration

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler

metadata:
  name: vpa-apache
  namespace: apache

spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: apache-deployment

  updatePolicy:
    updateMode: Auto
```

Explanation:

- Watches the `apache-deployment`.
- Monitors CPU and Memory usage.
- Automatically adjusts resource requests.
- Restarts Pods if necessary to apply the new values.

---

# Example Scenario

Initial Deployment:

```
CPU Request:
100m

Memory Request:
128Mi
```

Actual application usage:

```
CPU:
5m

Memory:
45Mi
```

VPA recommendation:

```
CPU:
10m

Memory:
64Mi
```

With `updateMode: Auto`:

1. VPA detects that the requests are much higher than needed.
2. The Updater evicts the Pod.
3. Deployment creates a new Pod.
4. New Pod starts with approximately:

```
CPU Request:
10m

Memory Request:
64Mi
```

This reduces wasted cluster resources.

---

# Another Example

Deployment:

```
Request CPU:
100m
```

Application usage:

```
250m CPU
```

VPA recommendation:

```
300m CPU
```

The new Pod starts with:

```
Request CPU:
300m
```

This provides enough CPU resources for the application.

---

# Difference Between Request and Limit

**Request**

- Minimum guaranteed resource.
- Used by the Kubernetes Scheduler to place Pods on nodes.

**Limit**

- Maximum resource the container can use.
- If exceeded:
  - CPU is throttled.
  - Memory overuse can lead to an Out Of Memory (OOM) kill.

---

# Does VPA Change Requests or Limits?

By default, VPA adjusts **resource requests**.

It can also manage limits if configured with a resource policy, but the most common use is adjusting requests.

---

# Installing VPA

Clone the autoscaler repository:

```bash
git clone https://github.com/kubernetes/autoscaler.git
```

Go to the VPA directory:

```bash
cd autoscaler/vertical-pod-autoscaler
```

Install VPA:

```bash
./hack/vpa-up.sh
```

Verify installation:

```bash
kubectl get pods -n kube-system
```

Expected VPA components:

```
vpa-admission-controller
vpa-recommender
vpa-updater
```

---

# Useful Commands

Create Namespace:

```bash
kubectl apply -f namespace.yml
```

Deploy Application:

```bash
kubectl apply -f deployment.yml
```

Create Service:

```bash
kubectl apply -f service.yml
```

Create VPA:

```bash
kubectl apply -f vpa.yml
```

View VPA:

```bash
kubectl get vpa -n apache
```

Describe VPA:

```bash
kubectl describe vpa vpa-apache -n apache
```

View Pod Resources:

```bash
kubectl describe pod <pod-name> -n apache
```

View Current Usage:

```bash
kubectl top pod -n apache
```

Watch Pods:

```bash
kubectl get pods -n apache -w
```

---

# HPA vs VPA

| Feature | HPA | VPA |
|----------|-----|-----|
| Full Name | Horizontal Pod Autoscaler | Vertical Pod Autoscaler |
| Scaling | Adds or removes Pods | Changes CPU/Memory of Pods |
| Uses Metrics | Yes | Yes |
| CPU Scaling | Yes | Yes |
| Memory Scaling | Yes | Yes |
| Restarts Pods | No | Sometimes (Auto mode) |
| Best For | Stateless applications | Applications needing dynamic resource sizing |

---

# Best Practices

- Install the Metrics Server before using VPA.
- Start with `updateMode: Off` to review recommendations.
- Use `Auto` only after validating recommendations.
- Avoid running VPA and HPA together on the same CPU or memory metrics, as they can conflict.
- Monitor recommendations before applying them in production.

---

# Summary

- VPA automatically adjusts CPU and Memory requests based on real usage.
- It helps optimize resource allocation and reduce waste.
- VPA relies on the Metrics Server for usage data.
- Main components: Recommender, Updater, and Admission Controller.
- `Auto` mode can evict Pods so that new Pods start with updated resource requests.
- VPA is ideal for applications whose resource requirements change over time.
