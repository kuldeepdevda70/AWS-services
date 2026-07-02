# Persistent Volume (PV) and Persistent Volume Claim (PVC)

## What is a Persistent Volume (PV)?

A **Persistent Volume (PV)** is a storage resource in the Kubernetes cluster that is created by the administrator. It provides permanent storage for applications.

### My PV Configuration

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
|-------|-------------|
| `kind: PersistentVolume` | Creates a Persistent Volume. |
| `apiVersion: v1` | Uses Kubernetes Core API. |
| `name: local-pv` | Name of the Persistent Volume. |
| `capacity.storage: 1Gi` | Total storage available is **1 GiB**. |
| `accessModes: ReadWriteOnce` | The volume can be mounted as read-write by only one node. |
| `persistentVolumeReclaimPolicy: Retain` | Keeps the data even after the PVC is deleted. |
| `storageClassName: local-storage` | Associates this PV with the `local-storage` StorageClass. |
| `hostPath: /mnt/data` | Uses the local directory `/mnt/data` on the node as storage. |

---

## What is a Persistent Volume Claim (PVC)?

A **Persistent Volume Claim (PVC)** is a request for storage made by an application. Kubernetes finds a matching PV and binds it to the PVC.

### My PVC Configuration

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
|-------|-------------|
| `kind: PersistentVolumeClaim` | Creates a Persistent Volume Claim. |
| `apiVersion: v1` | Uses Kubernetes Core API. |
| `name: local-pvc` | Name of the PVC. |
| `namespace: nginx` | PVC is created inside the `nginx` namespace. |
| `accessModes: ReadWriteOnce` | Requests read-write access from one node. |
| `resources.requests.storage: 1Gi` | Requests **1 GiB** of storage. |
| `storageClassName: local-storage` | Requests a PV with the `local-storage` StorageClass. |

---

# How PV and PVC Work Together

```
Application (Pod)
        │
        ▼
Persistent Volume Claim (PVC)
        │
        ▼
Persistent Volume (PV)
        │
        ▼
Host Directory
/mnt/data
```

---

# Matching Conditions

For a PVC to bind with a PV, the following must match:

- Storage Class (`local-storage`)
- Requested Storage (`1Gi`)
- Access Mode (`ReadWriteOnce`)

If all these values match, Kubernetes automatically binds the PVC to the PV.

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

Check PV

```bash
kubectl get pv
```

Check PVC

```bash
kubectl get pvc -n nginx
```

Describe PV

```bash
kubectl describe pv local-pv
```

Describe PVC

```bash
kubectl describe pvc local-pvc -n nginx
```

---

# Important Notes

- **PV** is the actual storage available in the cluster.
- **PVC** is a request for that storage.
- The Pod uses the **PVC**, not the PV directly.
- Kubernetes automatically connects the PVC to a matching PV.
- The storage is located at `/mnt/data` on the node because `hostPath` is used.
- Since the reclaim policy is `Retain`, deleting the PVC does **not** delete the stored data.
