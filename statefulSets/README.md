# StatefulSets in Kubernetes

## Overview

A **StatefulSet** is a Kubernetes workload resource used to manage applications that require stable pod identities, predictable network identities, or persistent storage.

Unlike a typical Deployment, a StatefulSet assigns each pod a predictable name and ordinal number. When a pod is deleted or crashes, Kubernetes recreates it with the same identity.

StatefulSets are commonly used for:
- MySQL and PostgreSQL databases
- MongoDB
- Apache Kafka
- Distributed systems that require stable identities and storage

## 1. Deployment vs. StatefulSet

| Feature | Deployment | StatefulSet |
|---|---|---|
| Pod names | Generated dynamically | Predictable names |
| Pod identity | Replicas are generally interchangeable | Each pod has a stable identity |
| Pod recreation | Replacement may have a different name | Replacement retains the same name |
| Persistent storage | Supported with additional configuration | Supports per-pod storage through `volumeClaimTemplates` |
| Typical use case | Stateless applications | Applications requiring stable identity or storage |

### Example

A Deployment might create pods with names like:

```text
nginx-7d8f9c6b5-x2k4p
nginx-7d8f9c6b5-m8j2q
nginx-7d8f9c6b5-r5w7n
```

A StatefulSet named `mysql-statefulset` creates pods with predictable names:

```text
mysql-statefulset-0
mysql-statefulset-1
mysql-statefulset-2
```

If `mysql-statefulset-1` is deleted, Kubernetes recreates it with the same name.

**Important:** A stable pod identity does not automatically guarantee data persistence. Persistent storage must be configured separately.

## 2. Project Architecture

This project deploys three MySQL instances using a StatefulSet.

```text
StatefulSet: mysql-statefulset
          |
          ├── mysql-statefulset-0
          |        └── Persistent Volume Claim 0
          |
          ├── mysql-statefulset-1
          |        └── Persistent Volume Claim 1
          |
          └── mysql-statefulset-2
                   └── Persistent Volume Claim 2

Headless Service: mysql-service
          |
          └── Provides DNS discovery for individual pods
```

Each pod receives its own PersistentVolumeClaim (PVC) through `volumeClaimTemplates`.

These three MySQL instances are independent databases. A StatefulSet does not automatically configure replication or turn them into a highly available MySQL cluster.

## 3. Prerequisites

Ensure you have:
- Linux
- Docker
- Minikube
- `kubectl` or the `k` alias
- A working Kubernetes cluster

Check your Minikube status:

```bash
minikube status
```

Check the available nodes:

```bash
k get nodes
```

## 4. Create the Namespace

Create a dedicated namespace for the MySQL resources:

```bash
k create namespace mysql-namespace
```

Verify:

```bash
k get namespaces
```

If the namespace already exists, you can continue without creating it again.

## 5. Create the Headless Service

Create a file named `service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql-service
  namespace: mysql-namespace
spec:
  clusterIP: None
  selector:
    app: mysql
  ports:
    - name: mysql
      protocol: TCP
      port: 3306
      targetPort: 3306
```

### Understanding the configuration

- `clusterIP: None`: Creates a headless Service.
- `selector`: Selects pods labeled `app: mysql`.
- `port: 3306`: Exposes the MySQL service port.
- `targetPort: 3306`: Forwards traffic to the container's MySQL port.

A headless Service allows clients to discover individual StatefulSet pods through DNS.

For example:

```text
mysql-statefulset-0.mysql-service.mysql-namespace.svc.cluster.local
mysql-statefulset-1.mysql-service.mysql-namespace.svc.cluster.local
mysql-statefulset-2.mysql-service.mysql-namespace.svc.cluster.local
```

These DNS names become usable when the corresponding pods and Service are configured and available.

## 6. Create the StatefulSet

Create a file named `statefulset.yaml`:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql-statefulset
  namespace: mysql-namespace
spec:
  serviceName: mysql-service
  replicas: 3

  selector:
    matchLabels:
      app: mysql

  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
        - name: mysql
          image: mysql:8.0
          imagePullPolicy: IfNotPresent

          ports:
            - containerPort: 3306

          env:
            - name: MYSQL_ROOT_PASSWORD
              value: root

            - name: MYSQL_DATABASE
              value: cloud

          volumeMounts:
            - name: mysql-data
              mountPath: /var/lib/mysql

  volumeClaimTemplates:
    - metadata:
        name: mysql-data
      spec:
        accessModes:
          - ReadWriteOnce
        resources:
          requests:
            storage: 1Gi
```

**Security note:** The password shown here is for local practice only. Use Kubernetes Secrets or an appropriate external secrets manager for real deployments.

### Understanding the configuration

| Field | Purpose |
|---|---|
| `serviceName` | Associates the StatefulSet with its governing Service |
| `replicas: 3` | Requests three MySQL pods |
| `matchLabels` | Defines which pods belong to the StatefulSet |
| `template` | Defines how each pod is created |
| `volumeMounts` | Mounts persistent storage inside the container |
| `volumeClaimTemplates` | Creates a separate PVC for each pod |
| `storage: 1Gi` | Requests 1 GiB of storage per pod |

## 7. Deploy the Resources

Apply the Service first:

```bash
k apply -f service.yaml
```

Apply the StatefulSet:

```bash
k apply -f statefulset.yaml
```

Check the Service:

```bash
k get service -n mysql-namespace
```

Check the StatefulSet:

```bash
k get statefulsets -n mysql-namespace
```

Check the pods:

```bash
k get pods -n mysql-namespace
```

Once all pods are ready, the output should resemble:

```text
NAME                   READY   STATUS    RESTARTS
mysql-statefulset-0    1/1     Running   0
mysql-statefulset-1    1/1     Running   0
mysql-statefulset-2    1/1     Running   0
```

MySQL initialization may take some time.

## 8. Understand Persistent Storage

Check the PersistentVolumeClaims:

```bash
k get pvc -n mysql-namespace
```

You should see three PVCs associated with the three pods.

The names follow this pattern:

```text
mysql-data-mysql-statefulset-0
mysql-data-mysql-statefulset-1
mysql-data-mysql-statefulset-2
```

Each pod has its own claim and can retain its database files when the pod is replaced, provided the underlying storage remains available and the PVC is retained.

## 9. Test StatefulSet Pod Recreation

This experiment demonstrates one of the most important StatefulSet behaviors.

### Step 1: List the pods

```bash
k get pods -n mysql-namespace
```

Example:

```text
mysql-statefulset-0
mysql-statefulset-1
mysql-statefulset-2
```

### Step 2: Delete one pod

Delete the second pod:

```bash
k delete pod mysql-statefulset-1 -n mysql-namespace
```

### Step 3: Watch Kubernetes recreate it

```bash
k get pods -n mysql-namespace -w
```

Kubernetes detects that the StatefulSet has fewer than three ready or existing replicas and creates a replacement.

The replacement pod will again be named:

```text
mysql-statefulset-1
```

### What did we learn?

- The pod is recreated with the same name.
- Its StatefulSet ordinal remains `1`.
- Its IP address may change.
- It can reconnect to its existing PVC.
- Its database files can survive the replacement.

The new pod is a new instance, even though it has the same name.

Press `Ctrl+C` to stop watching the pods.

## 10. Troubleshooting

### ImagePullBackOff

Check the pod events:

```bash
k describe pod mysql-statefulset-0 -n mysql-namespace
```

If the MySQL image is available in your host's Docker environment but not inside Minikube, load it:

```bash
minikube image load mysql:8.0
```

The configured `imagePullPolicy: IfNotPresent` allows Kubernetes to use the locally available image.

Check the pod again:

```bash
k get pods -n mysql-namespace
```

### Check MySQL logs

```bash
k logs mysql-statefulset-0 -n mysql-namespace
```

### Check persistent volume claims

```bash
k get pvc -n mysql-namespace
```

### Check pod details

```bash
k describe pod mysql-statefulset-0 -n mysql-namespace
```

## Key Takeaways

1. A StatefulSet manages applications that require stable identities or persistent storage.
2. StatefulSet pod names contain predictable ordinal numbers.
3. Kubernetes recreates deleted StatefulSet pods using the same identity.
4. Persistent storage must be configured separately to preserve data.
5. A headless Service supports DNS discovery of individual StatefulSet pods.
6. `volumeClaimTemplates` creates a separate PVC for each StatefulSet pod.
7. StatefulSets do not automatically configure database replication or high availability.

**Remember:** Deployments are generally suited to interchangeable stateless replicas, while StatefulSets are useful when each replica needs a stable identity and potentially its own persistent storage.
