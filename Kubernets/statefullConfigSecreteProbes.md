# Kubernetes StatefulSet with ConfigMap and Secret (MySQL)

## Project Structure

```
mysql/
│── namespace.yml
│── config.yml
│── secret.yml
│── service.yml
└── statefulSet.yml
```

---

# What are we creating?

We are deploying a **MySQL database** in Kubernetes.

Since a database stores important data, we use a **StatefulSet** instead of a Deployment.

The StatefulSet uses:

- Namespace
- ConfigMap
- Secret
- Headless Service
- Persistent Storage

---

# Architecture

```text
                  Namespace (mysql)
                         │
                         ▼
                StatefulSet (mysql)
                         │
        ┌────────────────┼────────────────┐
        ▼                ▼                ▼
     mysql-0         mysql-1          mysql-2
        │                │                │
        ▼                ▼                ▼
      PVC              PVC              PVC
        │                │                │
        ▼                ▼                ▼
   Persistent Data  Persistent Data  Persistent Data

            ▲
            │
      Reads Environment Variables
            │
     ┌──────────────┐
     │              │
     ▼              ▼
 ConfigMap       Secret
(Database Name) (Root Password)

            ▲
            │
      Headless Service
     (mysql-service)
```

---

# Step 1 : Namespace

File

```yaml
namespace.yml
```

```yaml
kind: Namespace
apiVersion: v1

metadata:
  name: mysql
```

### Why?

Instead of creating everything in the default namespace, we create our own namespace.

Everything below will be created inside

```
mysql
```

---

# Step 2 : ConfigMap

File

```yaml
config.yml
```

```yaml
kind: ConfigMap
apiVersion: v1

metadata:
  name: configmap-server
  namespace: mysql

data:
  MYSQL-DATABASE: devops
```

## What is ConfigMap?

ConfigMap stores **normal configuration values**.

Examples

- Database Name
- App Name
- Port
- Environment

It should **NOT** store passwords.

Here it stores

```
MYSQL_DATABASE = devops
```

Later the StatefulSet reads this value.

---

# Step 3 : Secret

File

```yaml
secret.yml
```

```yaml
kind: Secret
apiVersion: v1

metadata:
  name: app-secret
  namespace: mysql

data:
  MYSQL_ROOT_PASSWORD: cm9vdAo=
```

## What is Secret?

Secret stores **sensitive information**

Examples

- Password
- API Keys
- Tokens
- Certificates

Here

```
MYSQL_ROOT_PASSWORD
```

is stored in Base64 format.

```
cm9vdAo=
```

After decoding

```
root
```

---

# Difference between ConfigMap and Secret

| ConfigMap | Secret |
|-----------|---------|
| Stores normal data | Stores sensitive data |
| Plain text | Base64 encoded |
| Database Name | Password |
| Environment Variables | API Keys |

Example

ConfigMap

```
MYSQL_DATABASE=devops
```

Secret

```
MYSQL_ROOT_PASSWORD=root
```

---

# Step 4 : Headless Service

File

```yaml
service.yml
```

```yaml
kind: Service
apiVersion: v1

metadata:
  name: mysql-service
  namespace: mysql

spec:
  clusterIP: None

  selector:
    app: mysql

  ports:
    - port: 3306
```

Notice

```yaml
clusterIP: None
```

This makes it a **Headless Service**.

---

## Why Headless Service?

A normal Service gives **one IP**.

StatefulSet needs to communicate with **each Pod individually**.

Therefore Kubernetes creates DNS names like

```
mysql-0.mysql-service.mysql.svc.cluster.local

mysql-1.mysql-service.mysql.svc.cluster.local

mysql-2.mysql-service.mysql.svc.cluster.local
```

Every pod gets its own identity.

---

# Step 5 : StatefulSet

File

```yaml
statefulSet.yml
```

---

## Basic Information

```yaml
kind: StatefulSet
```

We are creating a StatefulSet.

---

## Service Name

```yaml
serviceName: mysql-service
```

This connects the StatefulSet with the Headless Service.

Without it,

Pods cannot receive stable DNS names.

---

## Replicas

```yaml
replicas: 3
```

Three MySQL Pods will be created.

```
mysql-0

mysql-1

mysql-2
```

Notice

Deployment creates names like

```
mysql-8dfd78

mysql-ab234

mysql-xy912
```

StatefulSet creates

```
mysql-0

mysql-1

mysql-2
```

These names never change.

---

## Selector

```yaml
selector:
  matchLabels:
    app: mysql
```

The StatefulSet controls every Pod having

```
app=mysql
```

---

## Pod Labels

```yaml
template:
  metadata:
    labels:
      app: mysql
```

The Pods receive

```
app=mysql
```

The Service uses this label to find Pods.

---

# MySQL Container

```yaml
containers:
- name: mysql
  image: mysql:8.0
```

Creates the MySQL container.

---

# Container Port

```yaml
ports:
- containerPort: 3306
```

MySQL listens on

```
3306
```

---

# Reading Password from Secret

```yaml
env:
- name: MYSQL_ROOT_PASSWORD
```

Instead of writing

```yaml
value: root
```

we use

```yaml
valueFrom:
  secretKeyRef:
```

Kubernetes reads

```
Secret
↓

app-secret

↓

MYSQL_ROOT_PASSWORD

↓

root
```

Container finally gets

```
MYSQL_ROOT_PASSWORD=root
```

---

# Reading Database Name from ConfigMap

```yaml
env:
- name: MYSQL_DATABASE
```

Reads

```
ConfigMap

↓

configmap-server

↓

MYSQL-DATABASE

↓

devops
```

Container receives

```
MYSQL_DATABASE=devops
```

---

# Volume Mount

```yaml
volumeMounts:
- name: mysql-data
  mountPath: /var/lib/mysql
```

MySQL stores data inside

```
/var/lib/mysql
```

Instead of container storage,

it stores data inside a Persistent Volume.

---

# VolumeClaimTemplate

```yaml
volumeClaimTemplates:
```

This is the most important feature of StatefulSet.

Every Pod gets its own PVC.

```
mysql-0
    │
    ▼
PVC-0

mysql-1
    │
    ▼
PVC-1

mysql-2
    │
    ▼
PVC-2
```

Every Pod has its own storage.

---

# Why StatefulSet?

Imagine

```
mysql-1
```

crashes.

Kubernetes recreates

```
mysql-1
```

NOT

```
mysql-5
```

It keeps

- same name
- same DNS
- same storage

The database data is still there.

---

# Deployment vs StatefulSet

| Deployment | StatefulSet |
|------------|-------------|
| Stateless Apps | Stateful Apps |
| Random Pod Names | Fixed Pod Names |
| Shared Identity | Unique Identity |
| Shared Storage | Dedicated Storage |
| Web App | MySQL |
| Nginx | MongoDB |
| React | PostgreSQL |

---

# Complete Flow

```text
Namespace
     │
     ▼
ConfigMap --------┐
                  │
                  ▼
             StatefulSet
                  ▲
                  │
Secret -----------┘
                  │
                  ▼
          Creates 3 Pods
                  │
      ┌───────────┼───────────┐
      ▼           ▼           ▼
   mysql-0     mysql-1     mysql-2
      │           │           │
      ▼           ▼           ▼
     PVC         PVC         PVC
      │           │           │
      ▼           ▼           ▼
 Persistent   Persistent   Persistent
   Storage      Storage      Storage

                  ▲
                  │
         Headless Service
```

---

# Apply Everything

```bash
kubectl apply -f namespace.yml

kubectl apply -f config.yml

kubectl apply -f secret.yml

kubectl apply -f service.yml

kubectl apply -f statefulSet.yml
```

---

# Verify

Check Pods

```bash
kubectl get pods -n mysql
```

Check Service

```bash
kubectl get svc -n mysql
```

Check ConfigMap

```bash
kubectl get configmap -n mysql
```

Check Secret

```bash
kubectl get secret -n mysql
```

Check PVC

```bash
kubectl get pvc -n mysql
```

---

# Key Interview Questions

### Why do we use StatefulSet?

Because databases need stable identities and persistent storage.

---

### Why not Deployment?

Deployment creates random Pod names and does not guarantee stable storage.

---

### Why Headless Service?

To provide each StatefulSet Pod with its own DNS name.

---

### Why ConfigMap?

To store non-sensitive configuration such as the database name.

---

### Why Secret?

To securely store sensitive information such as passwords.

---

### Why volumeClaimTemplates?

To automatically create a dedicated PersistentVolumeClaim for each StatefulSet Pod.

---

# Summary

- Namespace isolates resources.
- ConfigMap stores configuration.
- Secret stores passwords.
- Headless Service provides stable DNS names.
- StatefulSet creates Pods with fixed identities.
- Each Pod gets its own Persistent Volume.
- StatefulSet is the preferred choice for databases like MySQL.





# Kubernetes Interview Notes (StatefulSet, ConfigMap, Secret)

---

# What is a StatefulSet?

## Interview Definition

> **StatefulSet is a Kubernetes workload resource used to deploy stateful applications like MySQL, PostgreSQL, MongoDB, Cassandra, and Kafka. It provides stable pod names, stable network identities, and dedicated persistent storage for every pod.**

---

## How to Explain in an Interview

> StatefulSet is used when an application needs to store data. Unlike a Deployment, it creates pods with fixed names such as `mysql-0`, `mysql-1`, and `mysql-2`. If a pod crashes, Kubernetes recreates it with the same name and attaches the same storage, so the application can continue working without losing its data.

---

## Why do we use StatefulSet?

- Provides stable Pod names.
- Provides stable DNS names.
- Each Pod gets its own Persistent Volume.
- Pods are created in order.
- Pods are deleted in reverse order.
- Suitable for stateful applications.

---

## Real-world Examples

- MySQL
- PostgreSQL
- MongoDB
- Cassandra
- Kafka
- ZooKeeper

---

## Interview Question

### Why did you use StatefulSet instead of Deployment?

**Answer**

> I used StatefulSet because MySQL is a stateful application. It needs stable pod identities and persistent storage. Deployment creates pods with random names and does not guarantee dedicated storage, whereas StatefulSet recreates the pod with the same name and attaches the same Persistent Volume after a restart.

---

# What is a ConfigMap?

## Interview Definition

> **A ConfigMap is a Kubernetes object used to store non-sensitive configuration data separately from the application. It allows applications to read configuration without rebuilding the Docker image.**

---

## How to Explain in an Interview

> Instead of hardcoding configuration values inside the application, we store them in a ConfigMap. The application can read these values as environment variables or mounted files.

---

## Why do we use ConfigMap?

- Separates configuration from application code.
- Easy to update configuration.
- No need to rebuild Docker images.
- Reusable across applications.

---

## Examples

- Database Name
- Application Name
- Environment
- Port Number
- Log Level

---

## Project Example

```yaml
data:
  MYSQL_DATABASE: devops
```

StatefulSet reads this value using:

```yaml
env:
- name: MYSQL_DATABASE
  valueFrom:
    configMapKeyRef:
      name: configmap-server
      key: MYSQL_DATABASE
```

The container receives:

```
MYSQL_DATABASE=devops
```

---

## Interview Question

### Why did you use ConfigMap?

**Answer**

> I used ConfigMap to store the MySQL database name because it is not sensitive information. This keeps configuration separate from the application and makes it easier to update later.

---

# What is a Secret?

## Interview Definition

> **A Secret is a Kubernetes object used to store sensitive information such as passwords, API keys, tokens, and certificates.**

---

## How to Explain in an Interview

> Instead of writing passwords directly in the YAML file, we store them inside a Secret. The application securely reads the Secret through environment variables or mounted files.

---

## Why do we use Secret?

- Protect passwords.
- Store API Keys.
- Store Tokens.
- Store Certificates.
- Keep sensitive data separate.

---

## Examples

- Database Password
- JWT Secret
- API Key
- OAuth Token
- TLS Certificate

---

## Project Example

```yaml
data:
  MYSQL_ROOT_PASSWORD: cm9vdAo=
```

StatefulSet reads it using:

```yaml
env:
- name: MYSQL_ROOT_PASSWORD
  valueFrom:
    secretKeyRef:
      name: app-secret
      key: MYSQL_ROOT_PASSWORD
```

The container receives:

```
MYSQL_ROOT_PASSWORD=root
```

---

## Interview Question

### Why did you use Secret?

**Answer**

> I used Secret because the MySQL root password is sensitive information. Keeping it in a Secret is more secure than hardcoding it inside the application or YAML file.

---

# ConfigMap vs Secret

| ConfigMap | Secret |
|------------|---------|
| Stores normal configuration | Stores sensitive data |
| Plain text | Base64 encoded |
| Database Name | Password |
| Port Number | API Key |
| Environment | Token |

---

# What is a Headless Service?

## Interview Definition

> **A Headless Service is a Kubernetes Service with `clusterIP: None`. Instead of assigning one virtual IP, Kubernetes creates a separate DNS entry for every Pod.**

---

## How to Explain in an Interview

> StatefulSet requires every Pod to have a unique identity. A Headless Service provides a stable DNS name for every Pod so they can communicate individually.

Example:

```
mysql-0.mysql-service.mysql.svc.cluster.local

mysql-1.mysql-service.mysql.svc.cluster.local

mysql-2.mysql-service.mysql.svc.cluster.local
```

---

## Why do we use a Headless Service?

- Stable DNS names.
- Pod-to-Pod communication.
- Required for StatefulSet.

---

# What is volumeMount?

## Interview Definition

> **A volumeMount attaches a Kubernetes Volume to a specific directory inside a container.**

---

## How to Explain in an Interview

> MySQL stores its data inside `/var/lib/mysql`. The volumeMount connects this directory to persistent storage so that data remains available even if the container restarts.

Example:

```yaml
volumeMounts:
- name: mysql-data
  mountPath: /var/lib/mysql
```

---

# What is volumeClaimTemplates?

## Interview Definition

> **volumeClaimTemplates automatically create a separate PersistentVolumeClaim (PVC) for every StatefulSet Pod.**

---

## How to Explain in an Interview

If we create:

```yaml
replicas: 3
```

Kubernetes automatically creates:

```
mysql-0
   │
   ▼
PVC-0

mysql-1
   │
   ▼
PVC-1

mysql-2
   │
   ▼
PVC-2
```

Each Pod gets its own dedicated storage.

---

# Deployment vs StatefulSet

| Deployment | StatefulSet |
|------------|-------------|
| Stateless applications | Stateful applications |
| Random Pod names | Fixed Pod names |
| No stable identity | Stable identity |
| Shared or temporary storage | Dedicated storage |
| Used for React, Node.js, Nginx | Used for MySQL, MongoDB |

---

# Explain Your Project (Interview Answer)

> I deployed a MySQL database using Kubernetes StatefulSet. First, I created a Namespace to isolate the resources. Then I created a ConfigMap to store the database name and a Secret to store the MySQL root password. After that, I created a Headless Service because StatefulSet requires stable DNS names for each Pod. Finally, I deployed a StatefulSet with three replicas. Each Pod received a fixed name like `mysql-0`, `mysql-1`, and `mysql-2`, along with its own PersistentVolumeClaim created through `volumeClaimTemplates`. The MySQL container read the database name from the ConfigMap and the root password from the Secret using environment variables.

---

# Common Interview Questions

## 1. What is StatefulSet?

> StatefulSet is a Kubernetes resource used for stateful applications. It provides stable Pod names, stable DNS, and dedicated persistent storage.

---

## 2. Why not Deployment?

> Deployment is used for stateless applications. StatefulSet is used for databases because it preserves Pod identity and storage.

---

## 3. What is ConfigMap?

> ConfigMap stores non-sensitive configuration data separately from the application.

---

## 4. What is Secret?

> Secret stores sensitive information like passwords, API keys, and tokens.

---

## 5. Why Headless Service?

> Headless Service provides a unique DNS name for every StatefulSet Pod.

---

## 6. What is volumeMount?

> volumeMount connects a Kubernetes Volume to a directory inside a container.

---

## 7. What is volumeClaimTemplates?

> It automatically creates a dedicated PersistentVolumeClaim for every StatefulSet Pod.

---

# One-Minute Interview Answer

> StatefulSet is a Kubernetes workload resource used for stateful applications like MySQL. It provides stable Pod names, stable DNS names, and dedicated persistent storage for every Pod. In my project, I deployed MySQL using a StatefulSet with three replicas. I used a ConfigMap to store the database name, a Secret to store the MySQL root password, and a Headless Service to provide stable DNS names. Each Pod received its own PersistentVolumeClaim through `volumeClaimTemplates`, ensuring that data remains available even if a Pod is restarted.









# Kubernetes Probes (Short Interview Notes)

## What are Probes?

Probes are health checks used by Kubernetes to monitor the status of a container.

They help Kubernetes determine:
- Is the application started?
- Is the application alive?
- Is the application ready to receive traffic?

---

## Types of Probes

### 1. Startup Probe

**Purpose:** Checks if the application has started successfully.

- Used for slow-starting applications.
- If it fails, Kubernetes restarts the container.
- Until it succeeds, Liveness and Readiness probes do not run.

**Example:**
```yaml
startupProbe:
  httpGet:
    path: /
    port: 80
```

---

### 2. Liveness Probe

**Purpose:** Checks if the application is still running.

- If the application hangs or crashes, Kubernetes restarts the container.

**Example:**
```yaml
livenessProbe:
  httpGet:
    path: /
    port: 80
```

---

### 3. Readiness Probe

**Purpose:** Checks if the application is ready to receive traffic.

- If it fails, the pod is removed from the Service.
- The container is **not** restarted.

**Example:**
```yaml
readinessProbe:
  httpGet:
    path: /
    port: 80
```

---

## Difference

| Probe | Checks | On Failure |
|--------|--------|------------|
| Startup | App has started | Restart container |
| Liveness | App is alive | Restart container |
| Readiness | App is ready for traffic | Stop sending traffic |

---

## Interview Answer

**What are Kubernetes Probes?**

Kubernetes Probes are health checks used to monitor containers.

- **Startup Probe** → Checks if the application has started.
- **Liveness Probe** → Checks if the application is alive.
- **Readiness Probe** → Checks if the application is ready to receive traffic.

---

## Memory Trick

- **Startup = Started**
- **Liveness = Alive**
- **Readiness = Ready**
