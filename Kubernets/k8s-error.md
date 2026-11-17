
# Kubernetes Pod Errors – Complete Troubleshooting Guide

## 1. Understanding Pod Creation

When Kubernetes creates a Pod, it passes through multiple stages:

```text
Pod Created
    |
    v
Scheduling
    |
    v
Node Selected
    |
    v
Image Pull
    |
    v
Container Creation
    |
    v
Volume Mounting
    |
    v
Container Starts
    |
    v
Application Running
```

A failure at any stage can prevent the Pod from reaching `Running`.

---

# 2. Common Kubernetes Pod Errors

## 1. Pending

**Meaning:** Pod is created, but Kubernetes cannot schedule it or complete required setup.

**Common causes:**
- Insufficient CPU
- Insufficient memory
- Node selector mismatch
- Node taints without tolerations
- PVC unavailable
- Resource quota exceeded

**Example:**
```text
STATUS: Pending
```

**Troubleshooting:**
```bash
kubectl describe pod <pod-name> -n <namespace>
kubectl get nodes
kubectl get pvc -n <namespace>
```

---

## 2. FailedScheduling

**Meaning:** Kubernetes scheduler cannot find a suitable node for the Pod.

**Common errors:**
```text
0/3 nodes are available: 3 Insufficient cpu.
0/3 nodes are available: 3 Insufficient memory.
node(s) had taint that the pod didn't tolerate.
node(s) didn't match Pod's node affinity.
```

**Troubleshooting:**
```bash
kubectl describe pod <pod-name> -n <namespace>
kubectl describe nodes
```

---

## 3. ContainerCreating

**Meaning:** Pod is scheduled, but Kubernetes is preparing the container.

**Possible reasons:**
- Downloading container image
- Setting up networking
- Mounting volumes
- Mounting Secrets
- Mounting ConfigMaps
- Container runtime operations

**Example:**
```text
STATUS: ContainerCreating
```

**Troubleshooting:**
```bash
kubectl describe pod <pod-name> -n <namespace>
```

Check Events for the actual problem.

---

## 4. ErrImagePull

**Meaning:** Kubernetes tried to download the container image but failed.

**Common causes:**
- Incorrect image name
- Incorrect image tag
- Image does not exist
- Private registry authentication failure
- Network connectivity issue

**Possible errors:**
```text
manifest unknown
pull access denied
unauthorized: authentication required
no basic auth credentials
dial tcp: i/o timeout
x509: certificate signed by unknown authority
```

**Troubleshooting:**
```bash
kubectl describe pod <pod-name> -n <namespace>
kubectl get deployment <deployment-name> -n <namespace> -o yaml
```

---

## 5. ImagePullBackOff

**Meaning:** Kubernetes repeatedly failed to download the image and is waiting longer before retrying.

**Flow:**
```text
Image Pull Attempt
       |
       v
     Failed
       |
       v
   ErrImagePull
       |
       v
    Retry
       |
       v
ImagePullBackOff
```

**Common causes:**
- Wrong image tag
- Private registry authentication
- Registry unavailable
- Network issue

**Troubleshooting:**
```bash
kubectl describe pod <pod-name> -n <namespace>
```

---

## 6. CreateContainerConfigError

**Meaning:** Kubernetes cannot prepare the container because its configuration references something unavailable or invalid.

**Common causes:**
- Secret not found
- ConfigMap not found
- Invalid environment variable reference
- Incorrect volume configuration

**Example:**
```text
Error: secret "database-secret" not found
```

**Troubleshooting:**
```bash
kubectl get secrets -n <namespace>
kubectl get configmaps -n <namespace>
kubectl describe pod <pod-name> -n <namespace>
```

---

## 7. CreateContainerError

**Meaning:** Kubernetes/container runtime failed to create the container.

**Possible causes:**
- Invalid container configuration
- Runtime error
- Mount problem
- Invalid command or entrypoint

**Troubleshooting:**
```bash
kubectl describe pod <pod-name> -n <namespace>
```

---

## 8. CrashLoopBackOff

**Meaning:** Container starts but repeatedly crashes or exits. Kubernetes keeps restarting it.

**Flow:**
```text
Container Starts
       |
       v
Application Crashes
       |
       v
Container Exits
       |
       v
Kubernetes Restarts
       |
       v
Application Crashes Again
       |
       v
CrashLoopBackOff
```

**Common causes:**
- Application bug
- Wrong environment variables
- Database connection failure
- Incorrect startup command
- Missing configuration
- Liveness probe failure

**Troubleshooting:**
```bash
kubectl logs <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace> --previous
kubectl describe pod <pod-name> -n <namespace>
```

**Important:** Check `--previous` to see logs from the previously terminated container.

---

## 9. OOMKilled

**Meaning:** Container was killed because it exceeded its available memory limit.

**Example:**
```text
Last State: Terminated
Reason: OOMKilled
Exit Code: 137
```

**Example configuration:**
```yaml
resources:
  requests:
    memory: 128Mi
  limits:
    memory: 256Mi
```

If the container exceeds its memory limit, it can be killed.

**Troubleshooting:**
```bash
kubectl describe pod <pod-name> -n <namespace>
kubectl top pod <pod-name> -n <namespace>
kubectl top nodes
```

---

## 10. FailedMount

**Meaning:** Kubernetes cannot mount a volume inside the Pod.

**Common causes:**
- PVC does not exist
- StorageClass issue
- NFS unavailable
- CSI driver problem
- Permission issue

**Possible error:**
```text
MountVolume.SetUp failed
```

**Troubleshooting:**
```bash
kubectl describe pod <pod-name> -n <namespace>
kubectl get pvc -n <namespace>
kubectl get pv
kubectl get storageclass
```

---

## 11. FailedAttachVolume

**Meaning:** Kubernetes cannot attach the storage volume to the node.

**Common causes:**
- Cloud disk attachment failure
- Volume already attached elsewhere
- CSI controller issue
- Cloud provider API problem

**Common platforms:**
- AWS EBS
- Azure Disk
- Google Persistent Disk

**Troubleshooting:**
```bash
kubectl describe pod <pod-name> -n <namespace>
kubectl get volumeattachments
```

---

## 12. Init:Error

**Meaning:** An init container failed. The main application container cannot start until init containers complete successfully.

**Example:**
```yaml
initContainers:
  - name: database-check
    image: busybox
    command: ["sh", "-c", "exit 1"]
```

**Troubleshooting:**
```bash
kubectl logs <pod-name> -n <namespace> -c <init-container-name>
kubectl describe pod <pod-name> -n <namespace>
```

---

## 13. Init:CrashLoopBackOff

**Meaning:** An init container repeatedly starts and crashes.

The main application container waits until the init container succeeds.

**Troubleshooting:**
```bash
kubectl logs <pod-name> -n <namespace> -c <init-container-name> --previous
kubectl describe pod <pod-name> -n <namespace>
```

---

## 14. Command or Entrypoint Error

**Meaning:** Container runtime cannot execute the configured startup command.

**Example:**
```yaml
command:
  - /app/start.sh
```

But `/app/start.sh` does not exist inside the image.

**Possible error:**
```text
exec: "/app/start.sh": stat /app/start.sh: no such file or directory
```

**Troubleshooting:**
```bash
kubectl describe pod <pod-name> -n <namespace>
```

Check the Dockerfile and Deployment command.

---

## 15. NodeNotReady

**Meaning:** Kubernetes node is not healthy or cannot communicate properly with the control plane.

**Common causes:**
- Kubelet stopped
- Container runtime stopped
- Network failure
- DiskPressure
- MemoryPressure
- PIDPressure

**Troubleshooting:**
```bash
kubectl get nodes
kubectl describe node <node-name>
```

On the affected Linux node:
```bash
sudo systemctl status kubelet
sudo systemctl status containerd
df -h
free -h
```

---

# 3. Quick Comparison Table

| Status / Error | Meaning |
|---|---|
| Pending | Pod waiting for scheduling or setup |
| FailedScheduling | No suitable node available |
| ContainerCreating | Container setup in progress |
| ErrImagePull | Image download failed |
| ImagePullBackOff | Image download repeatedly failing |
| CreateContainerConfigError | Invalid or missing configuration |
| CreateContainerError | Container creation failed |
| CrashLoopBackOff | Container repeatedly crashes |
| OOMKilled | Container killed due to memory exhaustion |
| FailedMount | Volume mount failed |
| FailedAttachVolume | Volume attachment failed |
| Init:Error | Init container failed |
| Init:CrashLoopBackOff | Init container repeatedly crashes |
| NodeNotReady | Node is unhealthy or unavailable |

**Note:** These are not all Pod phases. Some are displayed Pod status reasons, container waiting reasons, termination reasons, or event reasons.

---

# 4. Most Important Troubleshooting Commands

### Check all Pods
```bash
kubectl get pods -A
```

### Check Pods in a namespace
```bash
kubectl get pods -n <namespace>
```

### Get detailed Pod information
```bash
kubectl describe pod <pod-name> -n <namespace>
```

### Check application logs
```bash
kubectl logs <pod-name> -n <namespace>
```

### Check previous container logs
```bash
kubectl logs <pod-name> -n <namespace> --previous
```

### Check Pod events
```bash
kubectl get events -n <namespace> --sort-by='.lastTimestamp'
```

### Check node status
```bash
kubectl get nodes -o wide
```

### Check node resources
```bash
kubectl top nodes
kubectl top pods -A
```

### Check PVC
```bash
kubectl get pvc -A
```

---

# 5. Complete Troubleshooting Flow

```text
                 POD CREATION
                      |
                      v
               POD SCHEDULING
                      |
             +--------+--------+
             |                 |
           Failed            Success
             |                 |
             v                 v
       Pending /         Container Setup
       FailedScheduling       |
                              v
                         Image Pull
                              |
                     +--------+--------+
                     |                 |
                   Failed            Success
                     |                 |
                     v                 v
             ErrImagePull /      Container Creation
             ImagePullBackOff          |
                                       v
                               Config / Secret /
                               Volume Validation
                                       |
                              +--------+--------+
                              |                 |
                            Failed            Success
                              |                 |
                              v                 v
                       CreateContainer     Container Starts
                       ConfigError                |
                                                  v
                                           Application Starts
                                                  |
                                         +--------+--------+
                                         |                 |
                                       Failed            Success
                                         |                 |
                                         v                 v
                                  CrashLoopBackOff      Running
```

---

# 6. Important Interview Questions

**Q1. What is the difference between ErrImagePull and ImagePullBackOff?**

ErrImagePull means Kubernetes failed to pull an image. ImagePullBackOff means Kubernetes is delaying retries after repeated image pull failures.

**Q2. What is the difference between CrashLoopBackOff and ImagePullBackOff?**

CrashLoopBackOff means the container starts but repeatedly crashes. ImagePullBackOff means Kubernetes cannot download the container image.

**Q3. How do you troubleshoot CrashLoopBackOff?**

First check:
```bash
kubectl logs <pod-name> --previous -n <namespace>
kubectl describe pod <pod-name> -n <namespace>
```

**Q4. What causes OOMKilled?**

The container exceeds its memory limit and is killed by the kernel.

**Q5. How do you troubleshoot FailedScheduling?**

Check Pod events, node resources, taints, tolerations, node selectors, and affinity rules.

**Q6. Why does a Pod remain in ContainerCreating?**

Possible reasons include image pulling, volume mounting, CNI networking, or container runtime problems.

---

# 7. Golden Rule for Kubernetes Troubleshooting

Always follow this sequence:

```text
1. kubectl get pods
          |
          v
2. kubectl describe pod
          |
          v
3. Check Events
          |
          v
4. kubectl logs
          |
          v
5. kubectl logs --previous
          |
          v
6. Identify Root Cause
          |
          v
7. Fix Deployment / Config / Image / Node
          |
          v
8. Verify Pod Status
```

**Remember:**

- Pod status tells you what is happening.
- Events tell you why Kubernetes is having a problem.
- Logs tell you why the application is failing.
- Describe gives you detailed Pod and container information.

The goal is not just to restart a failing Pod. The goal is to identify and fix the root cause.
