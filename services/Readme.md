# Kubernetes Services

## 1. What Is a Kubernetes Service?

A **Kubernetes Service** is an API resource that provides a stable network endpoint for accessing a group of Pods.

Pods are temporary resources. They can be recreated, replaced, or scaled, and their IP addresses can change. If another application connects directly to a Pod IP, that connection may break when the Pod is replaced.

A Service solves this problem by providing a stable endpoint and forwarding traffic to eligible backend Pods.

### Why Do We Need Services?

Imagine that we have two Nginx Pods:

```text
nginx-pod-1: 10.244.0.3
nginx-pod-2: 10.244.0.4
```

If an application connects directly to `10.244.0.3`, it depends on that particular Pod's IP address.

If the Pod is deleted and recreated, its IP address may change.

A Service provides a stable endpoint:

```text
                     nginx-service
                    ClusterIP: 10.107.131.221
                              |
                    ---------------------
                    |                   |
               Nginx Pod 1          Nginx Pod 2
               10.244.0.3           10.244.0.4
```

Applications can communicate with the Service instead of tracking individual Pod IP addresses.

The Service selects eligible Pods using labels and selectors.

**Key takeaway:** A Service provides stable access to Pods and distributes traffic among eligible backends.

---

## 2. Kubernetes Architecture: How Everything Connects

Before creating a Service, understand the resources involved.

### 2.1 Cluster

A Kubernetes cluster is a collection of machines that run containerized applications.

A cluster consists of:

- **Control plane:** Manages the cluster and makes scheduling and resource-management decisions.
- **Worker nodes:** Run application workloads in Pods.

In a Minikube learning environment, you can run a single-node cluster on your local machine.

Check your cluster:

```bash
k get nodes
```

Example:

```text
NAME       STATUS   ROLES           AGE   VERSION
minikube   Ready    control-plane   ...   ...
```

### 2.2 Namespace

A Namespace logically separates resources within a cluster.

For example:

```text
Kubernetes Cluster
|
|-- default
|   |-- Pods
|   |-- Deployments
|
|-- nginx
|   |-- nginx-deployment
|   |-- nginx-service
|   |-- Nginx Pods
|
|-- monitoring
    |-- Monitoring resources
```

Namespaces help organize applications, teams, and environments.

Create a Namespace:

```bash
k create namespace nginx
```

List Namespaces:

```bash
k get namespaces
```

List Pods in a specific Namespace:

```bash
k get pods -n nginx
```

List Pods across all Namespaces:

```bash
k get pods -A
```

**Important:** Namespaces provide logical separation, not complete security isolation by themselves.

### 2.3 Pod

A Pod is the smallest deployable unit in Kubernetes.

It contains one or more containers that share networking and can share storage.

For example, an Nginx Pod runs a container serving web content on port 80.

Check Pods:

```bash
k get pods -n nginx
```

Get additional information:

```bash
k get pods -n nginx -o wide
```

Inspect a Pod:

```bash
k describe pod <pod-name> -n nginx
```

Pods are not normally exposed as permanent network endpoints because they can be replaced.

### 2.4 Deployment

A Deployment manages a set of replicated Pods through a ReplicaSet.

You declare the desired number of replicas, and Kubernetes works to maintain that state.

For example:

```text
Deployment: nginx-deployment
Desired replicas: 2
          |
          v
     ReplicaSet
       /     \
      v       v
   Pod 1     Pod 2
```

If a managed Pod is deleted, the Deployment's controllers work to restore the desired replica count.

Inspect a Deployment:

```bash
k get deployments -n nginx
```

Inspect its details:

```bash
k describe deployment nginx-deployment -n nginx
```

Check ReplicaSets:

```bash
k get replicasets -n nginx
```

**Important:** A Deployment manages the lifecycle and desired replica count of Pods. A Service provides network access to eligible Pods. They have different responsibilities.

### 2.5 Service

A Service selects Pods using labels and provides a stable endpoint.

For example:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  namespace: nginx
spec:
  type: ClusterIP
  selector:
    app: nginx-label
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

Here:

- `metadata.name` identifies the Service.
- `metadata.namespace` places the Service in the `nginx` Namespace.
- `spec.type` specifies how the Service is exposed.
- `spec.selector` identifies eligible Pods by their labels.
- `port` defines the Service's listening port.
- `targetPort` specifies the destination port on the selected Pods.

The selector must match the labels on the actual Pods.

---

## 3. Understanding Labels and Selectors

Labels and selectors are essential to understanding how Services find Pods.

A label is a key-value pair attached to a Kubernetes resource.

Example Pod labels:

```yaml
labels:
  app: nginx-label
```

A Service can select Pods using:

```yaml
selector:
  app: nginx-label
```

The key and value must match.

### Example

Suppose the Pods have these labels:

```text
nginx-pod-1: app=nginx-label
nginx-pod-2: app=nginx-label
```

The following selector matches both Pods:

```yaml
selector:
  app: nginx-label
```

But this selector does not match either Pod:

```yaml
selector:
  app: nginx
```

The result is a Service without matching endpoints.

### Inspect Pod Labels

```bash
k get pods -n nginx --show-labels
```

### Inspect the Service Selector

```bash
k describe svc nginx-service -n nginx
```

Look for:

```text
Selector:  app=nginx-label
Endpoints: ...
```

**Teaching point:** A Deployment's name does not automatically determine its Pods' labels. Always inspect the actual labels rather than guessing.

---

## 4. Service Networking: Port vs. TargetPort

Consider this configuration:

```yaml
ports:
  - protocol: TCP
    port: 80
    targetPort: 80
```

The two port fields have different purposes.

| Field | Meaning |
|---|---|
| `port: 80` | The port exposed by the Service. |
| `targetPort: 80` | The port on the selected backend Pod that receives traffic. |
| `protocol: TCP` | The transport protocol used by this Service port. |

Suppose the Service exposes port 80 and forwards traffic to container port 8080:

```yaml
ports:
  - port: 80
    targetPort: 8080
```

Traffic follows this path:

```text
Client
  |
  | Service port 80
  v
Kubernetes Service
  |
  | Target port 8080
  v
Application container port 8080
```

The Service port and target port do not have to be the same.

**Important:** `targetPort` does not itself configure the application to listen on that port. The application must actually accept traffic on the destination port.

---

## 5. Types of Kubernetes Services

Kubernetes supports several Service types.

### 5.1 ClusterIP

`ClusterIP` is the default Service type.

It exposes a Service on an internal cluster IP, making it accessible to clients with suitable cluster network access.

Example:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  namespace: nginx
spec:
  type: ClusterIP
  selector:
    app: nginx-label
  ports:
    - port: 80
      targetPort: 80
```

Use ClusterIP when applications need to communicate internally.

Examples include:

- A frontend communicating with a backend.
- A backend communicating with an internal API.
- One application communicating with another application inside the cluster.

A ClusterIP Service is not normally directly accessible from outside the cluster.

### 5.2 NodePort

NodePort exposes a Service through a port on the cluster nodes.

Example:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-nodeport
  namespace: nginx
spec:
  type: NodePort
  selector:
    app: nginx-label
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080
```

The specified NodePort must be within the cluster's configured NodePort range. The usual default range is `30000–32767`.

Access generally follows this pattern:

```text
http://<node-ip>:30080
```

Actual reachability depends on the cluster environment, routing, and firewall configuration.

NodePort is useful for learning and certain development scenarios, but production applications often use other exposure patterns.

### 5.3 LoadBalancer

A LoadBalancer Service requests an external load balancer from a supporting infrastructure provider.

Example:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-loadbalancer
  namespace: nginx
spec:
  type: LoadBalancer
  selector:
    app: nginx-label
  ports:
    - port: 80
      targetPort: 80
```

The exact behavior depends on the cluster's infrastructure and load-balancer implementation.

On a local Minikube cluster, you may need additional support to obtain an externally reachable address.

### 5.4 ExternalName

An ExternalName Service maps a Service name to an external DNS name.

Example:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-api
  namespace: nginx
spec:
  type: ExternalName
  externalName: example.com
```

This is a DNS alias, not a conventional Service that selects Pods and forwards traffic to their endpoints.

### Service Types Summary

| Type | Main purpose |
|---|---|
| ClusterIP | Internal cluster access |
| NodePort | Access through a port on cluster nodes |
| LoadBalancer | External load-balancer integration |
| ExternalName | DNS alias for an external name |

---

## 6. Create a Complete Nginx Deployment and Service

This exercise creates a Namespace, a Deployment with two replicas, and a Service.

### Step 1: Create the Namespace

```bash
k create namespace nginx
```

If the Namespace already exists, inspect it instead of recreating it:

```bash
k get namespace nginx
```

### Step 2: Create `deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-label
  template:
    metadata:
      labels:
        app: nginx-label
    spec:
      containers:
        - name: nginx
          image: nginx:stable
          ports:
            - containerPort: 80
```

Apply it:

```bash
k apply -f deployment.yaml
```

Check the Deployment and Pods:

```bash
k get deployments -n nginx
k get replicasets -n nginx
k get pods -n nginx -o wide
```

### Step 3: Create `service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  namespace: nginx
spec:
  type: ClusterIP
  selector:
    app: nginx-label
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

Apply it:

```bash
k apply -f service.yaml
```

### Step 4: Verify the Service

```bash
k get services -n nginx
```

Inspect its configuration:

```bash
k describe service nginx-service -n nginx
```

Check its endpoints:

```bash
k get endpoints nginx-service -n nginx
```

On Kubernetes versions that support EndpointSlices, you can also inspect them:

```bash
k get endpointslices -n nginx
```

You should see endpoints for the ready Pods selected by the Service.

**Note:** EndpointSlices are the modern API for representing Service backends. The older Endpoints API is deprecated in Kubernetes v1.33 and later.

### Step 5: Test the Service from Inside the Cluster

Start a temporary Nginx client Pod:

```bash
k run nginx-client \
  -n nginx \
  --image=busybox:1.36 \
  --restart=Never \
  --command -- sleep 3600
```

Wait for it to become ready:

```bash
k wait --for=condition=Ready pod/nginx-client \
  -n nginx --timeout=120s
```

Send a request to the Service:

```bash
k exec -n nginx nginx-client -- \
  wget -qO- http://nginx-service
```

The Service name resolves through Kubernetes DNS within the Namespace.

Because the client Pod and Service are in the same Namespace, `nginx-service` is sufficient as the DNS name.

A fully qualified Service DNS name follows this general pattern:

```text
nginx-service.nginx.svc.cluster.local
```

The cluster DNS domain can be configured differently, so `cluster.local` is the common default rather than a universal requirement.

### Step 6: Access Nginx from Your Local Browser

Use port-forwarding:

```bash
k port-forward service/nginx-service \
  -n nginx 8080:80 \
  --address=127.0.0.1
```

Expected output:

```text
Forwarding from 127.0.0.1:8080 -> 80
```

Open:

```text
http://localhost:8080
```

Port-forwarding creates a temporary local connection to the Service. It does not change the Service's type or make its ClusterIP externally accessible.

Keep the terminal running while testing.

### Step 7: Clean Up the Exercise

Remove the temporary client Pod:

```bash
k delete pod nginx-client -n nginx
```

If you want to remove the entire exercise, delete the Namespace:

```bash
k delete namespace nginx
```

**Warning:** Deleting the Namespace deletes the namespaced resources inside it, including the Deployment, ReplicaSets, Pods, and Service. Do not run this command if you need to keep those resources.

---

## 7. The Correct Kubernetes Workflow

When building an application, follow this general workflow:

```text
1. Choose or create a Namespace
             |
             v
2. Define the Deployment
             |
             v
3. Apply the Deployment
             |
             v
4. Kubernetes creates/manages ReplicaSets and Pods
             |
             v
5. Confirm that Pods are ready
             |
             v
6. Inspect Pod labels
             |
             v
7. Create a Service whose selector matches those labels
             |
             v
8. Verify Service endpoints or EndpointSlices
             |
             v
9. Test access from inside the cluster
             |
             v
10. Use port-forwarding or an appropriate exposure method
```

You do not always need to create resources in exactly this order. Kubernetes controllers continuously reconcile resources, so you can create a Service before its Pods are ready. However, following this sequence makes learning and troubleshooting easier.

### How the Resources Are Interconnected

```text
Namespace: nginx
|
|-- Deployment: nginx-deployment
|       |
|       v
|    ReplicaSet
|       |
|       |-- Pod 1
|       |    label: app=nginx-label
|       |
|       |-- Pod 2
|            label: app=nginx-label
|
|-- Service: nginx-service
        |
        | selector: app=nginx-label
        |
        v
     Matching, eligible Pod endpoints
```

The Deployment does not directly attach the Service to its Pods. The Service independently selects Pods using labels.

This distinction is fundamental to Kubernetes networking.

---

## 8. Troubleshooting Common Service Problems

### Problem 1: Service Has No Endpoints

Check:

```bash
k get pods -n nginx --show-labels
k describe svc nginx-service -n nginx
k get endpoints nginx-service -n nginx
```

If the Service selector is:

```yaml
selector:
  app: nginx
```

but the Pods have:

```text
app=nginx-label
```

the Service will not select those Pods.

Correct the selector:

```yaml
selector:
  app: nginx-label
```

Then apply the updated manifest:

```bash
k apply -f service.yaml
```

Verify:

```bash
k get endpoints nginx-service -n nginx
```

### Problem 2: Port-Forward Times Out

Example:

```text
error: timed out waiting for the condition
```

Check these resources in order:

```bash
k get pods -n nginx -o wide
k get svc nginx-service -n nginx
k get endpoints nginx-service -n nginx
k describe svc nginx-service -n nginx
```

Verify that:

1. The backend Pods are running and ready.
2. The Service selector matches the Pod labels.
3. The Service has endpoints.
4. The Service's `targetPort` matches the port on which the application listens.

A missing endpoint is one possible cause, but not the only cause of a port-forward timeout.

### Problem 3: Service Exists but the Application Does Not Respond

Inspect the Pods:

```bash
k logs -n nginx -l app=nginx-label --tail=50
```

Inspect a specific Pod:

```bash
k describe pod <pod-name> -n nginx
```

Check whether the application listens on the expected port and whether network policies or other networking constraints interfere with communication.

### Problem 4: Port-Forward Says the Local Port Is Already in Use

Use another local port:

```bash
k port-forward service/nginx-service \
  -n nginx 8080:80 \
  --address=127.0.0.1
```

Here, local port `8080` forwards to Service port `80`.

The local port and Service port do not have to be identical.

### Problem 5: Service Selector Is Correct, but Endpoints Are Still Missing

Check Pod readiness:

```bash
k get pods -n nginx
k describe pods -n nginx
```

Inspect the EndpointSlices:

```bash
k get endpointslices -n nginx
```

A matching Pod that is not ready may not be included among the Service's ready endpoints under the default publishing behavior.

Also verify that the Service and selected Pods are in the same Namespace.

---

## 9. Essential Commands Cheat Sheet

Assuming your Namespace is `nginx`:

| Task | Command |
|---|---|
| List Namespaces | `k get namespaces` |
| List all Pods | `k get pods -A` |
| List Pods in the Namespace | `k get pods -n nginx` |
| Show Pod IPs and nodes | `k get pods -n nginx -o wide` |
| Show Pod labels | `k get pods -n nginx --show-labels` |
| List Deployments | `k get deployments -n nginx` |
| List ReplicaSets | `k get replicasets -n nginx` |
| List Services | `k get svc -n nginx` |
| Inspect a Service | `k describe svc nginx-service -n nginx` |
| List Endpoints | `k get endpoints nginx-service -n nginx` |
| List EndpointSlices | `k get endpointslices -n nginx` |
| View Pod logs | `k logs <pod-name> -n nginx` |
| Apply a manifest | `k apply -f service.yaml` |
| Delete a Service | `k delete svc nginx-service -n nginx` |
| Forward a Service port | `k port-forward svc/nginx-service -n nginx 8080:80` |

The `k` command in these examples is an alias for `kubectl`. If your terminal does not have this alias, use `kubectl` instead.

---

## 10. Best Practices for Learners

1. **Use Deployments for stateless application workloads.** Let the Deployment manage replica counts and replacement Pods.
2. **Use Services for stable network access.** Avoid relying on individual Pod IP addresses.
3. **Understand labels and selectors.** A Service's selector must match the intended backend Pods.
4. **Use Namespaces to organize resources.** Keep related resources together where appropriate.
5. **Verify endpoints before troubleshooting connectivity.** An empty endpoint list is an important diagnostic clue.
6. **Understand the distinction between internal and external access.** ClusterIP, NodePort, LoadBalancer, and port-forwarding serve different purposes.
7. **Apply declarative manifests.** Store resource configurations in YAML files so that the configuration can be reviewed, version-controlled, and reapplied.
8. **Inspect before changing.** Use `get`, `describe`, and `logs` to diagnose problems before editing manifests.

---

## 11. Practice Exercises

After completing this lesson, try these exercises:

1. Create a Namespace named `practice`.
2. Deploy two Nginx replicas in that Namespace.
3. Inspect the Pods and identify their labels.
4. Create a ClusterIP Service that selects both Pods.
5. Verify the Service endpoints.
6. Access the Service using a temporary client Pod.
7. Use port-forwarding to open Nginx in your browser.
8. Deliberately change the Service selector to a nonexistent label.
9. Observe the endpoints and investigate why the Service no longer selects the Pods.
10. Restore the correct selector and verify that connectivity works again.

### Questions to Test Your Understanding

- Why is a Service useful when Pod IP addresses can change?
- What is the difference between a Deployment and a Service?
- What role does a ReplicaSet play?
- Why must a Service selector match Pod labels?
- What is the difference between `port` and `targetPort`?
- Does creating a ClusterIP Service automatically expose an application to the internet?
- What does port-forwarding do?
- What is the difference between a Service and an EndpointSlice?

## Final Summary

Remember these four responsibilities:

- **Namespace:** Organizes and logically separates resources.
- **Deployment:** Manages the desired state of an application and its Pods through ReplicaSets.
- **Pod:** Runs the application's container or containers.
- **Service:** Provides a stable network endpoint and selects eligible backend Pods using labels.

Together, these resources form a fundamental part of how applications are deployed and accessed in Kubernetes.
