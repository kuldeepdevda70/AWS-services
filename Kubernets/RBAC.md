# Example: role.yml

```yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1

metadata:
  name: apache-manager
  namespace: apache

rules:
- apiGroups:
  - ""
  - apps
  - rbac.authorization.k8s.io
  - batch

  resources:
  - deployments
  - services
  - pods

  verbs:
  - get
  - delete
  - watch
  - create
  - patch
```

## Explanation

### kind: Role
Creates a Role object.

---

### apiVersion: rbac.authorization.k8s.io/v1
RBAC API version used by Kubernetes.

---

### metadata

```yaml
name: apache-manager
```

Role name.

```yaml
namespace: apache
```

This Role is valid only inside the **apache** namespace.

---

### rules

Defines what permissions this Role has.

---

### apiGroups

```yaml
- ""
- apps
- rbac.authorization.k8s.io
- batch
```

These are Kubernetes API groups.

| API Group | Resources |
|-----------|-----------|
| "" | Pods, Services, ConfigMaps, Secrets |
| apps | Deployments, ReplicaSets, StatefulSets |
| batch | Jobs, CronJobs |
| rbac.authorization.k8s.io | Roles, RoleBindings |

---

### resources

```yaml
resources:
- deployments
- services
- pods
```

These are the Kubernetes resources this Role can access.

---

### verbs

```yaml
verbs:
- get
- delete
- watch
- create
- patch
```

Allowed operations:

- get → Read
- create → Create
- delete → Delete
- watch → Monitor changes
- patch → Update part of an object

---

# Example: service-account.yml

```yaml
kind: ServiceAccount
apiVersion: v1

metadata:
  name: apache-user
  namespace: apache
```

## Explanation

Creates a ServiceAccount named **apache-user** in the **apache** namespace.

A ServiceAccount is an identity used by Pods to communicate with the Kubernetes API.

---

# Example: role-binding.yml

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding

metadata:
  name: apache-manager-binding
  namespace: apache

subjects:
- kind: User
  name: apache-user
  apiGroup: rbac.authorization.k8s.io

roleRef:
  kind: Role
  name: apache-manager
  apiGroup: rbac.authorization.k8s.io
```

## Explanation

### kind: RoleBinding

Assigns a Role to a User, Group, or ServiceAccount.

---

### metadata

```yaml
name: apache-manager-binding
```

RoleBinding name.

```yaml
namespace: apache
```

Works only inside the apache namespace.

---

### subjects

```yaml
kind: User
name: apache-user
```

The user who will receive the permissions.

---

### roleRef

```yaml
kind: Role
name: apache-manager
```

Assigns the **apache-manager** Role to **apache-user**.

Flow:

```
apache-user
      │
      ▼
RoleBinding
      │
      ▼
apache-manager Role
      │
      ▼
Permissions Granted
```

---

# Verify Permissions

```bash
kubectl auth can-i get pod
```

Output

```
yes
```

Current user has permission.

---

```bash
kubectl auth can-i get pod --as=apache-user
```

Output

```
no
```

No namespace is specified, so the Role in the `apache` namespace is not applied.

---

```bash
kubectl auth can-i get pod --as=apache-user -n apache
```

Output

```
yes
```

The Role is defined in the `apache` namespace, so `apache-user` has permission there.

---

# Interview Answer

"My Role named **apache-manager** is created in the **apache** namespace. It allows **get, create, delete, watch, and patch** operations on **pods, services, and deployments**. Then I created a **RoleBinding** named **apache-manager-binding**, which assigns this Role to **apache-user**. When I tested using `kubectl auth can-i`, the user had permissions only inside the **apache** namespace because Roles are namespace-scoped."
