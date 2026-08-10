# Terraform Infrastructure — EKS, ALB, RDS, CloudWatch

This document explains the Terraform setup used to provision AWS infrastructure (VPC, EKS, ALB, RDS, CloudWatch) with ArgoCD for GitOps-based application deployment. It's written so someone with no prior context on this code can read it top to bottom and understand what's built and why.

## Table of Contents
1. [Directory Structure](#directory-structure)
2. [Overall Architecture Flow](#overall-architecture-flow)
3. [Module Build Order (Dependency Chain)](#module-build-order-dependency-chain)
4. [VPC Module](#1-vpc-module)
5. [EKS Module](#2-eks-module)
6. [ALB Module](#3-alb-module)
7. [CloudWatch Module](#4-cloudwatch-module)
8. [RDS Module](#5-rds-module)
9. [ArgoCD](#6-argocd)
10. [End-to-End Request Flow](#end-to-end-request-flow)
11. [Things to Know / Revisit](#things-to-know--revisit)

---

## Directory Structure

```
.
├── argocd.tf
├── backend-sgp.yaml
├── default.conf
├── main.tf
├── module
│   ├── alb
│   │   ├── main.tf
│   │   ├── output.tf
│   │   └── variable.tf
│   ├── cloudwatch
│   │   ├── main.tf
│   │   ├── output.tf
│   │   └── variables.tf
│   ├── eks
│   │   ├── main.tf
│   │   ├── outputs.tf
│   │   └── variables.tf
│   ├── rds
│   │   ├── main.tf
│   │   ├── output.tf
│   │   └── variables.tf
│   └── vpc
│       ├── main.tf
│       ├── output.tf
│       └── variables.tf
├── output.tf
├── terraform.tfstate
├── terraform.tfstate.backup
├── terraform.tfvars
├── tfplan
└── variables.tf
```

Each folder under `module/` is a self-contained Terraform module. The root `main.tf` wires them together by passing outputs from one module in as inputs to another.

---

## Overall Architecture Flow

This is the big picture — how the five modules connect before diving into each one individually:

```
                         ┌─────────────────────────┐
                         │          VPC             │
                         │  public + private subnets│
                         └────────────┬──────────────┘
                                      │
              ┌───────────────────────┼───────────────────────┐
              │                       │                       │
   (public subnets)          (private subnets)         (private subnets)
              │                       │                       │
        ┌─────▼─────┐          ┌──────▼──────┐         ┌──────▼──────┐
        │    ALB     │◄────────│     EKS      │────────►│     RDS      │
        │ (internet  │  routes │  cluster +   │  reads/ │  (Postgres,  │
        │  facing)   │  traffic│  worker nodes│  writes │  private)    │
        └────────────┘  to     └──────┬───────┘         └─────────────┘
                                       │
                                       ▼
                                 ┌───────────┐
                                 │ CloudWatch │
                                 │   (logs)   │
                                 └───────────┘
                                       │
                                       ▼
                                 ┌───────────┐
                                 │  ArgoCD    │
                                 │ (GitOps —  │
                                 │ deploys to │
                                 │   EKS)     │
                                 └───────────┘
```

**In words:** the VPC is the network foundation everything else sits inside. EKS runs your application in worker nodes placed in private subnets. The ALB, in public subnets, is the only entry point from the internet and forwards traffic to those nodes. RDS is the database, reachable only by backend pods. CloudWatch collects logs from the cluster. ArgoCD watches a Git repo and keeps what's deployed on EKS in sync with it.

---

## Module Build Order (Dependency Chain)

Terraform builds in this order because each module needs outputs from the one before it:

```
1. VPC        → produces: vpc_id, public_subnet_ids, private_subnet_ids
2. EKS        → needs: private_subnet_ids
              → produces: cluster_name, cluster_security_group, node_group_asg_name
3. ALB        → needs: vpc_id, public_subnet_ids, node_group_asg_name, cluster_security_group
4. RDS        → needs: vpc_id, private_subnet_ids, cluster_security_group
5. CloudWatch → needs: cluster_name (logical grouping only, no hard resource dependency)
6. ArgoCD     → needs: EKS cluster to exist and be reachable
```

---

## 1. VPC Module

**Purpose:** the isolated network everything else lives inside.

| Component | Purpose |
|---|---|
| VPC | Defines the overall IP address range (CIDR block) |
| Public subnets | Reachable from the internet — for the ALB, NAT gateway |
| Private subnets | No direct internet access — for EKS nodes, RDS |
| Internet Gateway | Gives public subnets a path to/from the internet |
| NAT Gateway + Elastic IP | Lets private subnet resources reach the internet outbound (image pulls, updates) without being reachable inbound |
| Route Tables | Public route table points to the Internet Gateway; private route table points to the NAT Gateway |
| Route Table Associations | Bind each subnet to its correct route table |
| DNS settings | `enable_dns_support` + `enable_dns_hostnames` — required for EKS internal service discovery |

**Design requirements:**
- At least 2 Availability Zones — mandatory for EKS, and good practice generally
- Non-overlapping CIDR ranges for the VPC, public subnets, and private subnets
- Internet-facing components (ALB, NAT) go in public subnets; everything else (compute, DB) goes in private subnets

**What it outputs, and who uses it:**

| Output | Used by |
|---|---|
| `vpc_id` | ALB, EKS, RDS modules |
| `public_subnet_ids` | ALB module |
| `private_subnet_ids` | EKS node groups, RDS module |

---

## 2. EKS Module

**Purpose:** the managed Kubernetes cluster that runs your application.

### IAM Roles

Two separate roles are created, because the control plane and the worker nodes are different identities with different jobs — keeping them separate limits the damage if one is ever compromised.

- **`eks_cluster_role`** — the identity the EKS service itself uses to manage your cluster.
  - `AmazonEKSClusterPolicy` — lets it create and manage core control plane resources (API server, etcd, scheduler, controller manager, networking).
  - `AmazonEKSVPCResourceController` — lets it assign pod IP addresses from the VPC.

- **`eks_node_role`** — the identity the EC2 worker nodes use.
  - `AmazonEKSWorkerNodePolicy` — lets a node register itself into the cluster.
  - `AmazonEKS_CNI_Policy` — lets a node assign IP addresses to the pods running on it.
  - `AmazonEC2ContainerRegistryReadOnly` — lets a node pull container images from ECR.

### Cluster Configuration

- `subnet_ids` — set to `private_subnet_ids`, so the cluster's networking lives in private subnets.
- `endpoint_private_access` / `endpoint_public_access` — both `true`, so the API server is reachable both from inside the VPC and from the public internet (e.g. your local `kubectl`). Tightening this later to specific CIDRs is worth considering.
- `version` — pinned explicitly, so upgrades don't happen silently.
- `enabled_cluster_log_types` — conditionally streams control-plane logs (api, audit, authenticator, controller manager, scheduler) to CloudWatch, based on `enable_cluster_logging`.

### Node Group Configuration

- `subnet_ids` — same private subnets as the cluster, so worker nodes stay off the public internet.
- `capacity_type` — `ON_DEMAND` vs `SPOT`, a cost/interruption tradeoff.
- `scaling_config` — `desired/min/max_size` define the Auto Scaling Group's boundaries.
- `update_config.max_unavailable` — how many nodes can be down at once during a rolling update.
- `depends_on` the three node IAM policy attachments — explicit wait so nodes don't try to join before their permissions exist.

**What it outputs, and who uses it:**

| Output | Used by |
|---|---|
| `cluster_name` | CloudWatch, ArgoCD, kubectl/helm |
| `cluster_endpoint` | kubectl/helm, kubeconfig |
| `cluster_security_group` | ALB, RDS (for security group trust rules) |
| `node_group_asg_name` | ALB module (attaches ALB target group to this ASG) |
| `cluster_ca_certificate` | kubeconfig / kubernetes provider auth |

---

## 3. ALB Module

**Purpose:** the internet-facing "front door" that routes incoming traffic to healthy worker nodes.

1. **Security group for the ALB** — allows inbound traffic on the listener port from `ingress_cidr_blocks` (e.g. anywhere), allows all outbound.
2. **The ALB itself** — placed in public subnets, so it's reachable from the internet.
3. **Target group** — the list of "destinations" (worker nodes) the ALB can send traffic to, with a health check so it only sends traffic to nodes that are actually working.
4. **Listener** — "when traffic comes in on this port, forward it to that target group."
5. **Auto scaling attachment** — connects the EKS node group's Auto Scaling Group to the target group, so nodes are automatically registered/deregistered as they scale.
6. **Security group rule on the nodes** — the worker nodes only accept traffic from the ALB's security group, not directly from the internet.

**What it outputs:**

| Output | Purpose |
|---|---|
| `alb_dns_name` | The URL used to actually reach the application |
| `target_group_arn` | Reference for other resources needing the target group |

---

## 4. CloudWatch Module

**Purpose:** a place to store logs so you can look back and debug after the fact.

Three separate log groups are created, each auto-deleting entries after `retention_days` (30 by default):

1. **EKS cluster log group** — receives control plane logs (API activity, auth attempts, scheduling), fed by `enabled_cluster_log_types` in the EKS module.
2. **Application log group** — intended for the application's own logs.
3. **Container log group** — intended for logs from containers running on the nodes.

**Note:** creating these log groups doesn't automatically ship logs into them. The EKS cluster logs flow in on their own via `enabled_cluster_log_types`, but the application/container log groups need something like Fluent Bit or CloudWatch Container Insights actually running on the nodes to send logs there.

---

## 5. RDS Module

**Purpose:** a private PostgreSQL database that only backend pods can reach — not the public internet, not even the rest of the cluster.

1. **Fetch credentials from Secrets Manager** — username/password are pulled from a stored secret (`myapp/rds/credentials`) at deploy time instead of being written into the code.
2. **DB subnet group** — restricts RDS to launch only inside the private subnets from the VPC module.
3. **Security group for RDS** — one inbound rule: allow port `5432` (Postgres), but only from a specific security group (`backend_pod_sg`) — not an IP range, not the whole VPC.
4. **`backend_pod_sg`** — a separate security group with no inbound rules of its own, meant to be attached to backend pods. It exists purely as a "badge" the RDS security group recognizes as trusted.
5. **The database instance** — small Postgres instance (`db.t3.micro`, 20GB), not publicly accessible, single-AZ, using the credentials and networking from the steps above.

**Connection flow:**

```
Backend pod (has backend_pod_sg attached)
        │  sends request on port 5432
        ▼
RDS security group checks: "is this traffic from backend_pod_sg?"
        ├── Yes → allowed → reaches the Postgres instance
        └── No  → blocked
```

**Why this design:** this is security-group-to-security-group trust rather than IP-based trust, so it stays correct even as pod IPs change (which happens constantly in Kubernetes). The database has no public IP and no path from the internet.

**What it outputs:**

| Output | Purpose |
|---|---|
| `rds_endpoint` | Address the application uses to connect to the DB |
| `rds_port` | Port to connect on |
| `backend_pod_sg_id` | Attached to backend pods so they're trusted by the RDS security group |

---

## 6. ArgoCD

**Purpose:** installs ArgoCD onto the EKS cluster and points it at your Git repo, so deployments happen by pushing to Git rather than running `kubectl apply` manually.

### Connecting Terraform to the cluster

Before Terraform can create anything *inside* Kubernetes (namespaces, Helm charts, manifests), the `helm`, `kubernetes`, and `kubectl` providers need to know how to authenticate to the cluster. All three are configured the same way:

- `host` / `cluster_ca_certificate` — the cluster's API address and certificate, both pulled from the EKS module's outputs.
- `exec` block — instead of a static token, this runs `aws eks get-token --cluster-name ...` at apply time to generate a short-lived auth token. This avoids storing any long-lived credential for cluster access.

Three separate providers are configured because each does a different job: `kubernetes` manages plain K8s objects (like the namespace), `helm` installs Helm charts (ArgoCD itself), and `kubectl` applies raw YAML manifests (the ArgoCD Application object).

### Step-by-step

1. **`kubernetes_namespace.argocd`** — creates the `argocd` namespace where ArgoCD itself will live.

2. **`helm_release.argocd`** — installs ArgoCD into the cluster using its official Helm chart, into the namespace created above. `depends_on = [module.eks]` makes sure the cluster fully exists before Helm tries to talk to it.

3. **`kubectl_manifest.three_tier_app`** — creates an ArgoCD **Application** resource, which is what tells ArgoCD *what to deploy and where*:
   - `source` — where to pull manifests from: this GitHub repo, `main` branch, root path.
   - `destination` — where to deploy them: the `3-tier-app-eks` namespace, on the same cluster ArgoCD is running on (`https://kubernetes.default.svc`).
   - `syncPolicy.automated` — `prune: true` removes resources from the cluster if they're removed from Git; `selfHeal: true` automatically reverts any manual changes made directly in the cluster back to match Git.
   - `syncOptions: CreateNamespace=true` — creates the `3-tier-app-eks` namespace automatically if it doesn't already exist.
   - `depends_on = [helm_release.argocd]` — waits for ArgoCD itself to be installed before registering an application with it.

### What this achieves

Once this runs, ArgoCD is watching the `eks-terraform-application` repo continuously. Any change pushed to `main` — updated image tag, new manifest, changed config — gets automatically applied to the `3-tier-app-eks` namespace on the cluster, and any manual `kubectl` change made directly against the cluster gets automatically reverted back to match Git (because of `selfHeal`).

```
Git repo (main branch)
      │  ArgoCD polls/watches for changes
      ▼
ArgoCD Application (three-tier-app)
      │  syncs automatically (prune + selfHeal)
      ▼
3-tier-app-eks namespace on EKS
```

---

## End-to-End Request Flow

Putting every module together, here's what happens when a user hits your application:

```
User request
    │
    ▼
ALB (public subnet) — checks which nodes are healthy
    │
    ▼
Target Group → forwards to a healthy EKS worker node (private subnet)
    │
    ▼
Pod running on that node handles the request
    │
    ├── needs data? → connects to RDS (private subnet, only via backend_pod_sg)
    │
    └── logs the activity → CloudWatch log groups
```

Meanwhile, separately: ArgoCD continuously checks Git and keeps the pods running on EKS in sync with what's committed — this is how new versions of the application get deployed.

---

## Things to Know / Revisit

- **`skip_final_snapshot = true`** on RDS — if the database is ever destroyed, no backup snapshot is taken automatically. Fine for dev/learning, but should be changed before storing real data.
- **`endpoint_public_access = true`** on EKS — the API server is reachable from the whole internet, not just your IP. Worth restricting via `public_access_cidrs` later.
- **Subnet tagging for the AWS Load Balancer Controller** — if using the controller for auto-discovery, public/private subnets typically need `kubernetes.io/role/elb = 1` and `kubernetes.io/role/internal-elb = 1` tags respectively.
- **State files** (`terraform.tfstate*`) and `tfplan` are present locally — these should be excluded via `.gitignore` and ideally migrated to a remote backend (e.g. S3 + DynamoDB) before team collaboration.
- **`backend-sgp.yaml`** and **`default.conf`** are application/service manifests, not Terraform code — consider moving them into a separate `manifests/` or `k8s/` folder for clarity.
