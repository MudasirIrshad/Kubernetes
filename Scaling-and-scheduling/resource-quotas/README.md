# Kubernetes Resources, Requests, Limits, LimitRange, and ResourceQuota

This guide explains how Kubernetes manages CPU and memory resources for Pods and containers. It includes practical YAML examples and commands that you can run in a Minikube or Kubernetes cluster.

## 1. Why Resource Management Matters

Containers need CPU and memory to run. If workloads are not given sensible resource settings, one application can consume too many resources and affect other applications running on the same node.

Kubernetes provides several mechanisms to manage resources:

- **Requests:** Specify the minimum resources a container needs for scheduling.
- **Limits:** Define the maximum resources a container can consume.
- **LimitRange:** Set default values and minimum/maximum resource constraints within a namespace.
- **ResourceQuota:** Restrict the total amount of resources that can be requested or consumed within a namespace.

> CPU and memory are compute resources. Persistent storage is managed separately through PersistentVolumes (PV) and PersistentVolumeClaims (PVC).

## 2. Understanding CPU and Memory Units

### CPU

Kubernetes measures CPU in cores. You can also specify CPU in millicores (`m`).

| Value | Meaning |
|---|---|
| `1` | 1 CPU core |
| `500m` | 0.5 CPU core |
| `200m` | 0.2 CPU core |
| `100m` | 0.1 CPU core |

For example, `100m` means one-tenth of a CPU core.

### Memory

Memory can be specified using decimal or binary units. Binary units such as `Mi` and `Gi` are commonly used in Kubernetes.

| Value | Meaning |
|---|---|
| `128Mi` | 128 mebibytes |
| `256Mi` | 256 mebibytes |
| `1Gi` | 1 gibibyte |
| `2Gi` | 2 gibibytes |

Use `Mi` and `Gi` for memory values to avoid ambiguity.

## 3. Resource Requests

A **request** tells Kubernetes how much CPU or memory a container needs.

The Kubernetes scheduler uses resource requests to determine whether a node has enough available resources to schedule a Pod.

### Example

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
    - name: nginx
      image: nginx:latest
      resources:
        requests:
          cpu: "100m"
          memory: "128Mi"
```

In this example:

- The container requests `100m` CPU.
- The container requests `128Mi` memory.
- Kubernetes considers these values when selecting a node for the Pod.
- The container can use more than its requested resources if resources are available and no limit prevents it.

**Important:** A request is not a hard upper limit on resource consumption.

## 4. Resource Limits

A **limit** defines the maximum amount of CPU or memory a container is allowed to consume.

### Example

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
    - name: nginx
      image: nginx:latest
      resources:
        requests:
          cpu: "100m"
          memory: "128Mi"
        limits:
          cpu: "200m"
          memory: "256Mi"
```

Here:

- CPU request: `100m`
- CPU limit: `200m`
- Memory request: `128Mi`
- Memory limit: `256Mi`

The container requests a baseline of resources but can consume more, up to its configured limits.

### What happens when a container reaches its limit?

CPU and memory behave differently.

| Resource | Behavior |
|---|---|
| CPU | The container is throttled when it tries to use more CPU than its limit. |
| Memory | The container may be terminated with an `OOMKilled` reason if it exceeds its memory limit. |

**CPU throttling:** Kubernetes restricts the CPU time available to the container.

**OOMKilled:** The Linux kernel terminates a process when it cannot satisfy memory requirements within the applicable memory constraints.

A memory limit is particularly important because an application that requires more memory than its limit may crash.

## 5. Requests and Limits Together

A common configuration uses lower requests and higher limits:

```yaml
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "200m"
    memory: "256Mi"
```

This configuration:

- Gives Kubernetes a baseline for scheduling.
- Allows the container to use additional resources when available.
- Prevents the container from consuming unlimited CPU or memory.

Requests and limits should be selected according to the application's actual resource requirements.

## 6. Kubernetes QoS Classes

Kubernetes assigns every Pod a Quality of Service (QoS) class based on its containers' CPU and memory requests and limits.

There are three QoS classes:

1. Guaranteed
2. Burstable
3. BestEffort

### 6.1 Guaranteed

A Pod receives the `Guaranteed` QoS class when every container has CPU and memory requests and limits, and each request equals its corresponding limit.

Example:

```yaml
resources:
  requests:
    cpu: "500m"
    memory: "256Mi"
  limits:
    cpu: "500m"
    memory: "256Mi"
```

Both CPU and memory requests are equal to their limits.

### 6.2 Burstable

A Pod receives the `Burstable` QoS class when it does not qualify as Guaranteed, but at least one container has a CPU or memory request or limit.

For example:

```yaml
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "200m"
    memory: "256Mi"
```

The requests are lower than the limits, so the Pod belongs to the Burstable class.

### 6.3 BestEffort

A Pod receives the `BestEffort` QoS class when none of its containers specifies CPU or memory requests or limits.

Example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: best-effort-pod
spec:
  containers:
    - name: nginx
      image: nginx:latest
```

No CPU or memory resources are configured.

### Check a Pod's QoS class

```bash
kubectl describe pod nginx-pod
```

Look for:

```text
QoS Class: Burstable
```

You can also use:

```bash
kubectl get pod nginx-pod -o jsonpath='{.status.qosClass}{"\n"}'
```

## 7. Applying Resource Settings to a Deployment

Resource settings are normally specified inside the container template of a Deployment.

Create a file named `deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: nginx
spec:
  replicas: 1
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
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "200m"
              memory: "256Mi"
```

Create the namespace and deploy the application:

```bash
kubectl create namespace nginx
kubectl apply -f deployment.yaml
```

Check the Deployment and Pods:

```bash
kubectl get deployments -n nginx
kubectl get pods -n nginx
kubectl describe pod <pod-name> -n nginx
```

Replace `<pod-name>` with the actual Pod name.

## 8. LimitRange

A **LimitRange** applies default resource values and constraints to individual containers or Pods within a namespace.

It can:

- Set default CPU and memory requests.
- Set default CPU and memory limits.
- Enforce minimum resource values.
- Enforce maximum resource values.

### Example: Creating a LimitRange

Create a file named `limitrange.yaml`:

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: container-resource-limits
  namespace: nginx
spec:
  limits:
    - type: Container
      default:
        cpu: "500m"
        memory: "512Mi"
      defaultRequest:
        cpu: "100m"
        memory: "128Mi"
      min:
        cpu: "50m"
        memory: "64Mi"
      max:
        cpu: "1"
        memory: "1Gi"
```

Apply it:

```bash
kubectl apply -f limitrange.yaml
```

Inspect it:

```bash
kubectl get limitrange -n nginx
kubectl describe limitrange container-resource-limits -n nginx
```

### Understanding the configuration

| Configuration | Purpose |
|---|---|
| `default` | Default limits applied when a container does not specify limits. |
| `defaultRequest` | Default requests applied when a container does not specify requests. |
| `min` | Minimum allowed resource values. |
| `max` | Maximum allowed resource values. |

For example:

- If a container does not specify a CPU limit, Kubernetes applies the default CPU limit of `500m`.
- If a container does not specify a memory request, Kubernetes applies the default memory request of `128Mi`.
- A container cannot specify a CPU limit above `1` core.
- A container cannot specify a memory limit above `1Gi`.

**Important:** A LimitRange does not limit the total resources used by an entire namespace. It applies constraints to individual objects.

For namespace-wide resource limits, use ResourceQuota.

## 9. ResourceQuota

A **ResourceQuota** limits the aggregate resource requests and limits across applicable objects in a namespace.

For example, you can restrict a namespace to:

- A total of `2` CPU cores in CPU requests.
- A total of `2Gi` in memory requests.
- A total of `4` CPU cores in CPU limits.
- A total of `4Gi` in memory limits.
- A maximum of `10` Pods.

### Example: Creating a ResourceQuota

Create a file named `resourcequota.yaml`:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: nginx
spec:
  hard:
    requests.cpu: "2"
    requests.memory: "2Gi"
    limits.cpu: "4"
    limits.memory: "4Gi"
    pods: "10"
```

Apply it:

```bash
kubectl apply -f resourcequota.yaml
```

Inspect the quota:

```bash
kubectl get resourcequota -n nginx
kubectl describe resourcequota compute-quota -n nginx
```

The `describe` output displays the hard limits and current resource usage.

### What happens when a quota is exceeded?

Suppose a namespace has a CPU request quota of `2` cores.

If existing workloads already consume the entire request quota, Kubernetes rejects a new Pod if its CPU request would exceed the available quota.

You may receive an error similar to:

```text
exceeded quota: compute-quota
```

The existing Pods are not automatically terminated just because a new workload would exceed the quota.

### ResourceQuota and resource requests

When a namespace has quotas for CPU or memory requests and limits, new Pods generally need to specify the corresponding resource requests and limits.

A LimitRange can provide default values for containers that omit them.

This is why LimitRange and ResourceQuota are often used together.

## 10. LimitRange vs ResourceQuota

Although both are used for resource management, they solve different problems.

| Feature | LimitRange | ResourceQuota |
|---|---|---|
| Scope | Individual containers or Pods | Entire namespace |
| Sets default requests | Yes | No |
| Sets default limits | Yes | No |
| Enforces minimum values | Yes | No |
| Enforces maximum per-container values | Yes | No |
| Limits total namespace CPU/memory | No | Yes |
| Can limit number of Pods | No | Yes |

### When should you use them together?

Use both when you want to:

- Ensure every container has sensible default resource values.
- Prevent individual containers from requesting excessive resources.
- Prevent a namespace from consuming more than its allocated resource budget.

**LimitRange controls individual workloads. ResourceQuota controls the namespace-wide budget.**

## 11. Useful Kubernetes Commands

### Inspect Pods and resource configurations

```bash
# List Pods
kubectl get pods -n nginx

# View detailed Pod information
kubectl describe pod <pod-name> -n nginx

# Check the QoS class
kubectl get pod <pod-name> -n nginx -o jsonpath='{.status.qosClass}{"\n"}'
```

### Inspect LimitRanges

```bash
# List LimitRanges
kubectl get limitrange -n nginx

# Describe a LimitRange
kubectl describe limitrange container-resource-limits -n nginx
```

### Inspect ResourceQuotas

```bash
# List ResourceQuotas
kubectl get resourcequota -n nginx

# Describe a ResourceQuota
kubectl describe resourcequota compute-quota -n nginx
```

### Check actual resource usage

```bash
# Show Pod resource usage
kubectl top pods -n nginx

# Show node resource usage
kubectl top nodes
```

> `kubectl top` requires the Kubernetes Metrics Server to be installed and working. It displays observed resource usage, not the requests and limits configured in the Pod specification.

## 12. Troubleshooting

### 12.1 Pod is Pending

If a Pod remains in the `Pending` state, inspect it:

```bash
kubectl describe pod <pod-name> -n nginx
```

Look at the **Events** section near the bottom.

Possible reasons include:

| Event or issue | Meaning |
|---|---|
| `Insufficient cpu` | The node does not have enough unallocated CPU to satisfy the Pod's request. |
| `Insufficient memory` | The node does not have enough unallocated memory to satisfy the Pod's request. |
| PVC-related events | The Pod's PersistentVolumeClaim is not bound. |
| Scheduling constraints | Node selectors, taints, affinity rules, or unavailable nodes may prevent scheduling. |

Do not assume every Pending Pod is caused by CPU or memory. Always inspect the Events section to identify the actual reason.

### 12.2 Container is OOMKilled

Inspect the Pod:

```bash
kubectl describe pod <pod-name> -n nginx
```

Look for `OOMKilled` in the container's last state or restart information.

Possible causes include:

- The memory limit is too low.
- The application has a memory leak.
- The application requires more memory during peak workloads.

Review the application's memory requirements before increasing the limit.

### 12.3 ResourceQuota rejects a Pod

Inspect the namespace quota:

```bash
kubectl describe resourcequota -n nginx
```

Check current usage against the hard limit.

Also verify that the Pod specifies the required resource requests and limits, or that a LimitRange provides appropriate defaults.

### 12.4 Understand namespaces and resource names

A namespace is different from a Deployment or Pod name.

```bash
# List namespaces
kubectl get namespaces

# List Deployments in a namespace
kubectl get deployments -n nginx

# List Pods in a namespace
kubectl get pods -n nginx
```

PersistentVolumes are cluster-scoped, while PersistentVolumeClaims are namespaced.

```bash
# List all PersistentVolumes
kubectl get pv

# List PersistentVolumeClaims in nginx namespace
kubectl get pvc -n nginx
```

## 13. Key Takeaways

1. **Requests** help Kubernetes decide where a Pod can be scheduled.
2. **Limits** define the maximum CPU and memory a container can consume.
3. CPU limit enforcement normally causes throttling.
4. Exceeding a memory limit can result in `OOMKilled`.
5. Kubernetes assigns each Pod a QoS class: `Guaranteed`, `Burstable`, or `BestEffort`.
6. **LimitRange** sets defaults and per-object minimum/maximum constraints within a namespace.
7. **ResourceQuota** controls aggregate resource consumption within a namespace.
8. LimitRange and ResourceQuota are often used together.
9. Always inspect Pod Events when troubleshooting a Pending Pod.
10. CPU and memory resource management is separate from PersistentVolume and PersistentVolumeClaim storage management.

## 14. Hands-on Practice

Try these exercises in your own test namespace:

1. Create a Deployment with a CPU request of `100m` and a memory request of `128Mi`.
2. Set CPU and memory limits higher than the requests.
3. Inspect the Pod and identify its QoS class.
4. Change requests to equal limits and observe the QoS class.
5. Remove all CPU and memory settings and observe the QoS class again.
6. Create a LimitRange and deploy a container without explicit resource settings. Inspect the resulting Pod specification.
7. Create a ResourceQuota and try deploying enough replicas to exceed its Pod or CPU quota.
8. Use `kubectl describe` to investigate why Kubernetes accepts or rejects each workload.

### Cleanup

When you finish practicing, you can delete the test namespace:

```bash
kubectl delete namespace nginx
```

**Warning:** This deletes all namespaced resources inside `nginx`, including Deployments, Pods, Services, and PersistentVolumeClaims. Only run it if you no longer need those resources.
