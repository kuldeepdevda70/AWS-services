# Kubernetes Persistent Volume (PV), Persistent Volume Claim (PVC), and Deployment

## Architecture

```
                +----------------------+
                |      Deployment      |
                |    (nginx Pods)      |
                +----------+-----------+
                           |
                           | Uses
                           ▼
                +----------------------+
                | Persistent Volume    |
                | Claim (local-pvc)    |
                +----------+-----------+
                           |
                           | Binds to
                           ▼
                +----------------------+
                | Persistent Volume    |
                |     (local-pv)       |
                +----------+-----------+
                           |
                           | Stores data in
                           ▼
                     /mnt/data
                     (Host Machine)
```

---

# 1. Persistent Volume (PV)

A **Persistent Volume (PV)** is the actual storage available in the Kubernetes cluster. It is created by the cluster administrator and provides storage for applications.

## Configuration

```yaml
kind: PersistentVolume
apiVersion: v1

metadata:
  name: local-pv

spec:
  capacity:
    storage: 1Gi

  accessModes:
    - ReadWriteOnce

  persistentVolumeReclaimPolicy: Retain

  storageClassName: local-storage

  hostPath:
    path: /mnt/data
```

### Explanation

| Field | Description |
|--------|-------------|
| `capacity.storage` | Total storage available (1Gi). |
| `accessModes` | ReadWriteOnce means one node can mount it as read-write. |
| `storageClassName` | Storage class used for matching with PVC. |
| `hostPath` | Uses the host directory `/mnt/data` as storage. |
| `persistentVolumeReclaimPolicy: Retain` | Keeps the data even if the PVC is deleted. |

---

# 2. Persistent Volume Claim (PVC)

A **Persistent Volume Claim (PVC)** is a request for storage by an application.

## Configuration

```yaml
kind: PersistentVolumeClaim
apiVersion: v1

metadata:
  name: local-pvc
  namespace: nginx

spec:
  accessModes:
    - ReadWriteOnce

  resources:
    requests:
      storage: 1Gi

  storageClassName: local-storage
```

### Explanation

| Field | Description |
|--------|-------------|
| `storage: 1Gi` | Requests 1Gi of storage. |
| `accessModes` | Requests ReadWriteOnce access. |
| `storageClassName` | Must match the PV's storage class. |

When the request matches a PV, Kubernetes automatically binds the PVC to the PV.

---

# 3. Deployment

The Deployment creates two Nginx Pods and mounts the storage provided by the PVC.

## Configuration

```yaml
kind: Deployment
apiVersion: apps/v1

metadata:
  name: nginx-deployment
  namespace: nginx

spec:
  replicas: 2

  selector:
    matchLabels:
      app: nginx

  template:
    metadata:
      labels:
        app: nginx

    spec:
      containers:
        - name: nginx
          image: nginx:latest

          ports:
            - containerPort: 80

          volumeMounts:
            - name: my-volume
              mountPath: /var/www/html

      volumes:
        - name: my-volume
          persistentVolumeClaim:
            claimName: local-pvc
```

### Explanation

| Field | Description |
|--------|-------------|
| `replicas: 2` | Creates two Nginx Pods. |
| `volumeMounts` | Mounts the storage inside the container. |
| `mountPath` | Storage is available at `/var/www/html`. |
| `claimName` | Uses the PVC named `local-pvc`. |

---

# Complete Flow

```
Step 1
Administrator creates PV
        │
        ▼
+----------------------+
| local-pv             |
| 1Gi Storage          |
| /mnt/data            |
+----------------------+

        │

Step 2
Developer creates PVC
        │
        ▼
+----------------------+
| local-pvc            |
| Requests 1Gi         |
| local-storage        |
+----------------------+

        │
        │ Kubernetes automatically binds them
        ▼

+----------------------+
| local-pv             |
+----------------------+

        │

Step 3
Deployment uses PVC
        │
        ▼

+----------------------+
| nginx Pod            |
| /var/www/html        |
+----------------------+

        │
        ▼

Data is stored in

/mnt/data
(on the Kubernetes Node)
```

---

# Why Use PVC Instead of PV Directly?

Pods do **not** use a PV directly.

Instead:

```
Pod
 ↓
PVC
 ↓
PV
 ↓
Actual Storage
```

This makes applications independent of the underlying storage.

---

# Matching Requirements

A PVC binds to a PV only if these match:

- Storage Class (`local-storage`)
- Storage Size (`1Gi`)
- Access Mode (`ReadWriteOnce`)

---

# Commands

Create PV

```bash
kubectl apply -f persistentVolume.yml
```

Create PVC

```bash
kubectl apply -f persistentVolumeClaim.yml
```

Create Deployment

```bash
kubectl apply -f deployment.yml
```

Check PV

```bash
kubectl get pv
```

Check PVC

```bash
kubectl get pvc -n nginx
```

Check Pods

```bash
kubectl get pods -n nginx
```

Describe PV

```bash
kubectl describe pv local-pv
```

Describe PVC

```bash
kubectl describe pvc local-pvc -n nginx
```

Describe Deployment

```bash
kubectl describe deployment nginx-deployment -n nginx
```

---

# Interview Summary

- **Persistent Volume (PV):** The actual storage resource in the cluster.
- **Persistent Volume Claim (PVC):** A request for storage by an application.
- **Deployment:** Creates Pods that use the PVC.
- **volumeMounts:** Specifies where the storage is mounted inside the container.
- **hostPath:** Stores the data on the Kubernetes node at `/mnt/data`.
- **Retain Policy:** Keeps the data even after the PVC is deleted.
