# Kubernetes Introduction & Architecture

## Introduction to Kubernetes

**Kubernetes (K8s)** is an open-source container orchestration platform used to deploy, manage, scale, and maintain containerized applications.

Instead of manually managing individual containers, Kubernetes provides a platform for managing containerized workloads across one or more machines called **Nodes**.

Kubernetes can automatically:

- Deploy containerized applications
- Restart failed containers
- Scale applications up or down
- Provide networking between applications
- Expose applications to users
- Manage configuration and secrets
- Perform rolling updates and rollbacks
- Distribute workloads across available nodes

### Why Kubernetes?

With Docker, we can manually run containers:

```text
Docker
├── Django container
├── MySQL container
└── Nginx container
```

As the number of containers and machines increases, manually managing them becomes difficult.

Kubernetes adds an orchestration layer that manages these workloads:

```text
Kubernetes Cluster
│
├── Node
│   ├── Pod
│   └── Pod
│
├── Node
│   ├── Pod
│   └── Pod
│
└── Node
    └── Pod
```

Kubernetes continuously works to maintain the desired state of the applications running in the cluster.

---

# Kubernetes Architecture

A Kubernetes cluster is mainly divided into two parts:

1. **Control Plane**
2. **Worker Nodes**

## Kubernetes Cluster Architecture

![Kubernetes Cluster Architecture](https://kubernetes.io/images/docs/kubernetes-cluster-architecture.svg)

The diagram above shows the main components that make up a Kubernetes cluster.

---

## 1. Control Plane

The **Control Plane** manages the Kubernetes cluster.

It is responsible for making decisions about the cluster and maintaining the desired state of the applications.

The main Control Plane components are:

### API Server

The **kube-apiserver** is the main entry point to the Kubernetes cluster.

When we run commands such as:

```bash
kubectl get pods
kubectl create deployment nginx --image=nginx
kubectl delete pod my-pod
```

`kubectl` communicates with the **API Server**.

The API Server validates requests and provides the API through which Kubernetes components communicate.

---

### etcd

**etcd** is the distributed key-value store used by Kubernetes to store cluster state.

It stores information about things such as:

- Pods
- Deployments
- Services
- Configurations
- Secrets
- Cluster state

You can think of **etcd as the database of the Kubernetes Control Plane**.

---

### Scheduler

The **kube-scheduler** decides which Worker Node should run a newly created Pod.

For example:

```text
New Pod
   │
   ▼
Scheduler
   │
   ▼
Worker Node 2
   │
   ▼
Pod
```

The scheduler considers available resources and scheduling constraints when making this decision.

---

### Controller Manager

The **kube-controller-manager** runs Kubernetes controllers.

Controllers continuously compare the desired state with the actual state of the cluster.

For example, if a Deployment specifies:

```yaml
replicas: 3
```

but one Pod crashes:

```text
Desired State: 3 Pods
Actual State:  2 Pods
```

The controller detects the difference and works to create another Pod.

---

# 2. Worker Nodes

**Worker Nodes** are the machines where application workloads actually run.

A Worker Node typically contains:

- kubelet
- kube-proxy
- Container Runtime
- Pods

---

## kubelet

The **kubelet** is an agent that runs on every Worker Node.

Its job is to make sure that the Pods assigned to that node are running correctly.

The general flow is:

```text
Control Plane
      │
      ▼
"Run this Pod on Node 1"
      │
      ▼
   kubelet
      │
      ▼
Container Runtime
      │
      ▼
   Container
```

---

## kube-proxy

**kube-proxy** is responsible for implementing part of Kubernetes networking on each node.

It helps route network traffic to the appropriate Pods and Services.

A simplified flow is:

```text
Client
  │
  ▼
Service
  │
  ▼
Networking
  │
  ▼
Pod
```

---

## Container Runtime

The **Container Runtime** is responsible for actually running containers.

Modern Kubernetes commonly uses **containerd** or another CRI-compatible container runtime.

The relationship is:

```text
Kubernetes
    │
    ▼
 kubelet
    │
    ▼
Container Runtime
    │
    ▼
Container
```

---

# Pods

A **Pod** is the smallest deployable unit in Kubernetes.

A Pod contains one or more containers that share networking and storage resources.

For a simple application:

```text
Pod
└── Django Container
```

A more complex Pod can contain multiple containers:

```text
Pod
├── Application Container
└── Sidecar Container
```

Most beginner applications start with one container per Pod.

For example, we created our Django Pod using:

```bash
kubectl run notes-app-pod \
  -n notes-app \
  --image=django-notes-app-django:latest \
  --image-pull-policy=Never \
  --restart=Never
```

This creates:

```text
notes-app namespace
└── notes-app-pod
    └── Django container
```

---

# Kubernetes Cluster: Putting Everything Together

The overall architecture can be simplified as:

```text
                     Kubernetes Cluster
                            │
             ┌──────────────┴──────────────┐
             │                             │
        Control Plane                 Worker Node
             │                             │
       ┌─────┼─────┐                ┌──────┼──────┐
       │     │     │                │      │      │
   API Server etcd Scheduler      kubelet kube-proxy
                                      │
                              Container Runtime
                                      │
                                     Pods
                                      │
                                Applications
```

The key idea is:

> **The Control Plane manages the Kubernetes cluster, while Worker Nodes run the application workloads.**

---

# Kubernetes and Minikube

When using **Minikube** locally, the Kubernetes cluster can run inside a single VM or container.

Our local environment looks roughly like:

```text
Linux Machine
     │
     ▼
  Minikube
     │
     ▼
Kubernetes Node
     │
     ▼
notes-app-pod
     │
     ▼
Django Container
```

We can inspect the cluster using:

```bash
kubectl get nodes
```

View namespaces:

```bash
kubectl get namespaces
```

View Pods:

```bash
kubectl get pods -n notes-app
```

View detailed information about a Pod:

```bash
kubectl describe pod notes-app-pod -n notes-app
```

This allows us to connect the Kubernetes architecture concepts with the actual cluster running on our local machine.
