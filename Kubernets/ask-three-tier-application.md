# How Kubernetes Components Communicate (Deployment → Service → Ingress)

This document explains **how the components are connected** and **how a request travels through the cluster**, instead of defining what each resource is.

---

# Application Architecture

```text
                        Browser
                           |
                 http://localhost
                           |
                           |
                    NGINX Ingress
                           |
          -----------------|------------------
          |                                |
      Path = /                        Path = /api
          |                                |
          |                                |
Frontend Service                     API Service
          |                                |
          |                                |
Frontend Pods                      Backend Pods
                                           |
                                           |
                                     MongoDB Service
                                           |
                                           |
                                      MongoDB Pod
```

---

# Step 1 - Deployment Creates Pods

Frontend Deployment

```yaml
template:
  metadata:
    labels:
      role: frontend
```

creates Pods like

```text
Frontend Pod 1

role=frontend

Frontend Pod 2

role=frontend
```

Backend Deployment

```yaml
template:
  metadata:
    labels:
      role: api
```

creates

```text
Backend Pod 1

role=api

Backend Pod 2

role=api
```

Notice that Deployments never communicate with each other.

They only create Pods with labels.

---

# Step 2 - Service Connects to Pods

Frontend Service

```yaml
selector:
  role: frontend
```

Kubernetes continuously checks

```text
Does any Pod have

role=frontend ?
```

If YES

```text
Frontend Service

↓

Frontend Pod
```

Backend Service

```yaml
selector:
  role: api
```

Kubernetes checks

```text
role=api
```

If YES

```text
API Service

↓

Backend Pods
```

This connection is created automatically using labels.

No Pod IP is written anywhere.

---

# Step 3 - Service Builds Endpoints

Suppose Kubernetes creates

```text
Frontend Pod

10.244.1.10

Frontend Pod

10.244.2.15
```

Frontend Service automatically builds

```text
Frontend Service

↓

10.244.1.10:3000

10.244.2.15:3000
```

These are called Endpoints.

Ingress never talks directly to Pods.

Ingress always talks to the Service.

The Service chooses one of its endpoints.

---

# Step 4 - Ingress Never Talks to Pods

Ingress contains

```yaml
paths:

- path: /

  backend:

    service:

      name: frontend

- path: /api

  backend:

    service:

      name: api
```

Notice something important.

Ingress knows only

```text
frontend Service

api Service
```

It does NOT know

```text
10.244.1.10

10.244.2.15
```

Those Pod IPs are hidden by the Service.

---

# Step 5 - Browser Sends Request

User opens

```text
http://localhost
```

Browser sends

```text
GET /
```

NGINX receives

```text
GET /
```

Ingress checks

```text
Does URL start with / ?
```

YES

↓

Forward request to

```text
Frontend Service
```

Frontend Service selects one endpoint

```text
10.244.1.10
```

Frontend Pod receives request.

---

# Step 6 - Frontend Calls Backend

Frontend contains

```text
REACT_APP_BACKEND_URL

http://localhost/api/tasks
```

Browser sends

```text
GET

http://localhost/api/tasks
```

Again

Browser sends request to

```text
localhost
```

NOT

```text
api Service
```

---

# Step 7 - Ingress Checks Again

NGINX receives

```text
GET

/api/tasks
```

Ingress checks

```text
Does URL begin with

/api ?
```

YES

↓

Forward request to

```text
API Service
```

API Service checks endpoints

```text
Backend Pod 1

10.244.3.20

Backend Pod 2

10.244.2.18
```

Service selects one Pod.

Backend receives request.

---

# Step 8 - Backend Calls MongoDB

Backend uses

```text
mongodb-svc
```

Example

```text
mongodb://mongodb-svc:27017
```

Backend asks Kubernetes DNS

```text
Where is mongodb-svc ?
```

DNS replies

```text
ClusterIP

10.96.82.117
```

MongoDB Service receives request.

MongoDB Service forwards request to

```text
MongoDB Pod
```

---

# Complete Communication Flow

```text
Browser

↓

localhost

↓

Ingress

↓

Frontend Service

↓

Frontend Pod

↓

Browser sends

/api/tasks

↓

Ingress

↓

API Service

↓

Backend Pod

↓

MongoDB Service

↓

MongoDB Pod
```

---

# How Does Ingress Decide Where to Send Requests?

Ingress checks only two things.

## 1. Host

Example

```yaml
host: myapp.com
```

or

```yaml
host: admin.myapp.com
```

---

## 2. Path

Example

```yaml
path: /
```

or

```yaml
path: /api
```

Example

```yaml
- path: /
```

means

```text
Everything beginning with /

↓

Frontend Service
```

Example

```yaml
- path: /api
```

means

```text
Everything beginning with

/api

↓

API Service
```

---

# Multiple Backend Services

Suppose there are

```text
User Service

Order Service

Product Service

Payment Service
```

Ingress

```yaml
/api/users

↓

user-service

/api/orders

↓

order-service

/api/products

↓

product-service

/api/payments

↓

payment-service
```

Browser requests

```text
/api/orders
```

Ingress compares paths

```text
/api/users

No

/api/orders

YES
```

↓

Send request to

```text
order-service
```

Next request

```text
/api/products
```

↓

Send to

```text
product-service
```

---

# Why Can't Two Services Use the Same Path?

Wrong

```yaml
/api

↓

user-service

/api

↓

order-service
```

Ingress receives

```text
/api
```

Which Service should it choose?

It cannot decide.

Therefore each route must be unique.

Correct

```text
/api/users

/api/orders

/api/products

/api/payments
```

---

# Why Doesn't Browser Call api Service Directly?

Browser runs outside Kubernetes.

It does not know

```text
api

frontend

mongodb-svc
```

Those names exist only inside the Kubernetes cluster.

Browser only knows

```text
localhost

or

myapp.com
```

Ingress is the only public entry point.

Ingress knows where every Service is.

Services know where every Pod is.

Pods know how to process the request.

---

# Entire Request Flow

```text
               Browser
                   |
                   |
          http://localhost
                   |
                   |
             NGINX Ingress
             /            \
            /              \
           /                \
          /                  \
Frontend Service         API Service
          |                  |
          |                  |
Frontend Pods          Backend Pods
                             |
                             |
                       MongoDB Service
                             |
                             |
                        MongoDB Pod
```

# Connection Summary

```text
Deployment
      │
      ▼
Creates Pods
      │
      ▼
Service finds Pods using Labels
      │
      ▼
Service creates Endpoints
      │
      ▼
Ingress forwards requests to Services
      │
      ▼
Service forwards requests to Pods
      │
      ▼
Backend accesses MongoDB Service
      │
      ▼
MongoDB Service forwards to MongoDB Pod
```
