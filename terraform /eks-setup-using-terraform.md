# Terraform Infrastructure – Project Structure

This repository contains the Terraform configuration for provisioning and managing the AWS infrastructure (VPC, EKS, ALB, RDS, CloudWatch) along with ArgoCD setup for GitOps-based deployments.

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
├── terraform.tfstate.1785999925.backup
├── terraform.tfstate.backup
├── terraform.tfvars
├── tfplan
└── variables.tf
```

# VPC Setup with Terraform — Overview & Requirements

## Core Components Required

| Component | Purpose |
|---|---|
| **VPC** | The isolated network boundary — defines the overall IP address range (CIDR block) |
| **Subnets (Public)** | Segments of the VPC reachable from the internet — for load balancers, NAT gateways |
| **Subnets (Private)** | Segments with no direct internet access — for compute (EKS nodes), databases (RDS) |
| **Internet Gateway (IGW)** | Attached to the VPC; gives public subnets a path to/from the internet |
| **NAT Gateway + Elastic IP** | Sits in a public subnet; lets private subnet resources reach the internet outbound (updates, image pulls) without being exposed inbound |
| **Route Tables** | Define traffic rules — public route table points to IGW, private route table points to NAT Gateway |
| **Route Table Associations** | Bind each subnet to its correct route table (public subnets ↔ public RT, private ↔ private RT) |
| **DNS Settings** | `enable_dns_support` + `enable_dns_hostnames` — required for internal service discovery (critical for EKS) |

## Design Requirements to Keep in Mind

- **Multi-AZ**: at least 2 Availability Zones — mandatory for EKS and generally good practice for high availability.
- **CIDR planning**: non-overlapping ranges for VPC, public subnets, and private subnets — plan ahead if you'll peer with other VPCs later.
- **Public vs Private separation**: internet-facing components (ALB, NAT) in public subnets; everything else (compute, DB) in private subnets.
- **Tagging**: consistent `Name` tags, plus (for EKS) subnet tags like `kubernetes.io/role/elb` and `kubernetes.io/role/internal-elb` so the AWS Load Balancer Controller can auto-discover subnets.
- **Modularity**: wrap VPC logic in its own module with a single input object (like your `vpc_config`) — keeps it reusable across environments via `.tfvars`.

## Typical File Layout




# EKS Module — Overview & Requirements

## Core Components Required

| Component | Purpose |
|---|---|
| **EKS Cluster IAM Role** | Lets the EKS control plane itself assume permissions to manage AWS resources (ENIs, load balancers, etc.) |
| **EKS Cluster** | The managed Kubernetes control plane — API server, etcd, scheduler managed by AWS |
| **Node IAM Role** | Lets worker node EC2 instances join the cluster and pull images |
| **Managed Node Group** | The actual EC2 worker nodes that run your pods |
| **Cluster Logging (optional)** | Sends control plane logs (api, audit, authenticator, etc.) to CloudWatch |

## IAM Roles — Why Two Separate Roles

- **`eks_cluster_role`** — assumed by the `eks.amazonaws.com` service. Required policy: `AmazonEKSClusterPolicy`, plus `AmazonEKSVPCResourceController`
  - - `AmazonEKSClusterPolicy` — lets it create and manage core control plane resources (API server, etcd, scheduler, controller manager, networking).
   - `AmazonEKSVPCResourceController` — lets it assign pod IP addresses from the VPC.

- **`eks_node_role`** — assumed by `ec2.amazonaws.com` (the worker instances). Requires three policies:
  - `AmazonEKSWorkerNodePolicy` — lets nodes register with the cluster
  - `AmazonEKS_CNI_Policy` — lets the CNI plugin assign pod IPs
  - `AmazonEC2ContainerRegistryReadOnly` — lets nodes pull images from ECR

These are kept separate because the control plane and the worker nodes are different trust principals with different permission needs — merging them would over-privilege one or the other.

## Cluster Configuration Requirements

- **`subnet_ids`** — must span ≥2 AZs; here it's `private_subnet_ids`, so the control plane's elastic network interfaces land in private subnets (nodes talk to the API server privately).
- **`endpoint_private_access` / `endpoint_public_access`** — both `true` here, meaning the API server is reachable both from inside the VPC and from the public internet (e.g. your local `kubectl`). For tighter security, public access is often restricted to specific CIDRs via `public_access_cidrs` — worth considering later.
- **`version`** — pinned via `var.cluster_version` so upgrades are explicit, not automatic.
- **`enabled_cluster_log_types`** — conditionally enables control-plane log streaming to CloudWatch based on `enable_cluster_logging`; useful for debugging auth/API issues but has cost implications if left on permanently.

## Node Group Configuration Requirements

- **`subnet_ids`** — same private subnets as the cluster, keeping compute off the public internet.
- **`capacity_type`** — `ON_DEMAND` vs `SPOT`; affects cost/interruption tradeoffs.
- **`scaling_config`** — `desired/min/max_size` define the Auto Scaling Group boundaries for the node group.
- **`update_config.max_unavailable`** — controls rolling update behavior during node group upgrades (how many nodes can be down at once).
- **`depends_on`** — explicitly waits for the three node IAM policy attachments before creating the node group, since IAM propagation can lag and nodes will fail to join if permissions aren't ready.

## Dependency Chain

## ALB (Application Load Balancer)

- `aws_security_group.alb` — firewall for the ALB. Allows inbound traffic on the listener port from `ingress_cidr_blocks` (e.g. anywhere), allows all outbound.

- `aws_lb.this` — the ALB itself. Sits in public subnets, uses the security group above.

- `aws_lb_target_group.this` — group of targets (nodes) the ALB sends traffic to. Has a health check to know which targets are healthy.

- `aws_lb_listener.http` — listens on the ALB's port, forwards matching traffic to the target group.

- `aws_autoscaling_attachment.this` — connects the EKS node group's Auto Scaling Group to the target group, so new/removed nodes are automatically registered/deregistered.

- `aws_security_group_rule.node_ingress_from_alb` — opens the node port so only the ALB's security group can reach it (not the whole internet).
## ALB (Application Load Balancer) — In Plain Terms

The ALB is the "front door" of the application. It sits between the internet and your Kubernetes nodes, and its job is to receive incoming traffic and forward it to a healthy server.

**What's being set up, step by step:**

1. **A firewall for the ALB** — decides who's allowed to reach it (e.g. anyone, on a specific port) and lets it send traffic back out freely.

2. **The load balancer itself** — placed in the public subnets so it's reachable from the internet.

3. **A target group** — this is just a list of "destinations" (worker nodes) the ALB can send traffic to. It also runs health checks, so it only sends traffic to nodes that are actually working.

4. **A listener** — tells the ALB "when traffic comes in on this port, forward it to that target group."

5. **Auto-attachment to the node group** — as nodes are added or removed by auto scaling, they're automatically added to or removed from the target group — no manual step needed.

6. **A firewall rule on the nodes** — makes sure the worker nodes only accept traffic that's coming from the ALB, not directly from the internet.

**Overall flow:**

A user's request comes in from the internet → hits the ALB → the ALB checks which nodes are healthy → forwards the request to one of those nodes → the node runs the application and responds back through the same path.

**Why it matters:** without this, you'd either have to expose worker nodes directly to the internet (insecure) or manually manage which servers get traffic. The ALB automates traffic distribution and keeps the actual servers hidden behind it.

## CloudWatch — In Plain Terms

CloudWatch is where logs get stored so you can look back and see what happened. This module just creates three separate storage "buckets" (log groups) for different kinds of logs, each set to auto-delete old entries after a certain number of days.

**What's being set up:**

1. **EKS cluster log group** — stores logs from the EKS control plane itself (API server activity, authentication attempts, scheduling decisions). This is where `enable_cluster_logging` from the EKS module actually sends its data.

2. **Application log group** — a general space intended for your application's own logs.

3. **Container log group** — a space for logs coming from containers running on the nodes.

4. **Retention** — all three are set to auto-delete logs after `retention_days` (30 by default), so storage doesn't grow forever and cost stays predictable.

**Why it matters:** without this, you'd have no easy way to debug issues after the fact — pods crash, requests fail, and by default that information just disappears once the pod restarts. These log groups give you a place to look back and investigate.

**Note:** creating the log groups here doesn't automatically send logs into them — the EKS module's `enabled_cluster_log_types` handles the cluster logs, but the application/container log groups need something like Fluent Bit or the CloudWatch Container Insights agent actually running on the nodes to ship logs into them.
