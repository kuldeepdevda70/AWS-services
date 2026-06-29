## **Monolithic**
Monolithic: A single application where all components (UI, business logic, and database) are built and deployed together.

## **Microservices**
Microservices: An application split into multiple independent services, where each service performs a specific function and can be developed, deployed, and scaled separately.

## **Kubernets Architecture** 

<img width="1744" height="1042" alt="kubernets" src="https://github.com/user-attachments/assets/f8b3ecbe-2477-43d4-b09a-b7c332aa2838" />



## kubectl: 
It is the CLI (Command Line Interface) used to create, update, delete, and manage Kubernetes resources.

## Master Node (Control Plane) Components
1. API Server: Receives all Kubernetes requests and acts as the main entry point to the cluster.
2. etcd: Stores all cluster data and configuration in a key-value database.
3. Scheduler: Decides which worker node should run a newly created Pod.
4. Controller Manager: Monitors the cluster and ensures the desired state is maintained (e.g., recreates failed Pods).

## Worker Node Components
1. Kubelet: An agent that communicates with the master node and ensures Pods are running correctly on the worker node.
2. Container Runtime: Runs and manages containers (e.g., Docker, containerd)
3. Kube Proxy: Manages networking and routes traffic to the correct Pods within the cluster.

