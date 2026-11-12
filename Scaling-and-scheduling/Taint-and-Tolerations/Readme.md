# Taints and Tolerations in Kubernetes

## 1. What Are Taints and Tolerations?

Taints and tolerations control which Pods are allowed to run on Kubernetes nodes.

Think of a node as a restricted room:

- **Taint:** A restriction placed on a node, like a "Staff Only" sign.
- **Toleration:** Permission given to a Pod to enter a restricted room.

A taint prevents Pods without a matching toleration from being scheduled on that node.

A toleration allows a Pod to be scheduled on a node with a matching taint.

**Important:** A toleration does not force a Pod to run on a particular node. It only allows the Pod to use that node.

---

## 2. What Is a Taint?

A taint is applied to a Kubernetes node.

Syntax:

```bash
kubectl taint nodes <node-name> <key>=<value>:<effect>
```

Example:

```bash
kubectl taint nodes minikube prod=true:NoSchedule
```

This command applies a taint to the `minikube` node.

It means:

- Key: `prod`
- Value: `true`
- Effect: `NoSchedule`

Now, Kubernetes will not schedule new Pods on this node unless they have a matching toleration.

### Verify the taint

```bash
kubectl describe node minikube
```

Look for the `Taints` section:

```text
Taints: prod=true:NoSchedule
```

---

## 3. Types of Taint Effects

Kubernetes provides three taint effects.

| Effect | Description |
|---|---|
| `NoSchedule` | Prevents new Pods without matching tolerations from being scheduled on the node. |
| `PreferNoSchedule` | Kubernetes tries to avoid scheduling Pods without matching tolerations on the node, but may still schedule them. |
| `NoExecute` | Prevents new Pods without matching tolerations from being scheduled and evicts existing Pods that do not tolerate the taint. |

### Understanding NoExecute

`NoExecute` can also use `tolerationSeconds`.

For example:

```yaml
tolerations:
  - key: "prod"
    operator: "Equal"
    value: "true"
    effect: "NoExecute"
    tolerationSeconds: 60
```

This allows the Pod to tolerate the taint for 60 seconds. If the taint remains, the Pod can be evicted after that period.

---

## 4. What Is a Toleration?

A toleration is added to a Pod's YAML configuration.

It tells Kubernetes that the Pod is allowed to run on a node with a matching taint.

Example:

```yaml
tolerations:
  - key: "prod"
    operator: "Equal"
    value: "true"
    effect: "NoSchedule"
```

This toleration matches the following taint:

```text
prod=true:NoSchedule
```

The key, value, and effect must match.

### Understanding the operators

| Operator | Description |
|---|---|
| `Equal` | Matches when both the key and value match. |
| `Exists` | Matches based on the key's existence, regardless of its value. |

Example using `Exists`:

```yaml
tolerations:
  - key: "prod"
    operator: "Exists"
    effect: "NoSchedule"
```

This tolerates a `NoSchedule` taint with the key `prod`, regardless of its value.

---

## 5. Practical Example: Taints and Tolerations with Minikube

Let's create a multi-node Minikube cluster with a control-plane node and a worker node.

### Step 1: Check your nodes

```bash
kubectl get nodes
```

Example:

```text
NAME           STATUS   ROLES
minikube       Ready    control-plane
minikube-m02   Ready    <none>
```

The node with the `control-plane` role manages the cluster. The additional node acts as a worker.

A worker node may display `<none>` under `ROLES`. This does not mean it cannot run application Pods.

### Step 2: Apply a taint to the control-plane node

```bash
kubectl taint nodes minikube prod=true:NoSchedule
```

Verify it:

```bash
kubectl describe node minikube
```

You should see:

```text
Taints: prod=true:NoSchedule
```

### Step 3: Create a Pod without a toleration

Create a file named `pod.yml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  namespace: nginx
  labels:
    app: nginx-label
spec:
  containers:
    - name: nginx
      image: nginx:latest
      ports:
        - containerPort: 80
```

Apply the configuration:

```bash
kubectl apply -f pod.yml
```

Check the Pod:

```bash
kubectl get pods -n nginx
```

If both nodes have taints that the Pod cannot tolerate, you may see:

```text
NAME        READY   STATUS
nginx-pod   0/1     Pending
```

Check the reason:

```bash
kubectl describe pod nginx-pod -n nginx
```

You may see an event similar to:

```text
0/2 nodes are available: 2 node(s) had untolerated taint(s).
```

This means Kubernetes could not find a suitable node for the Pod.

### Step 4: Add a toleration to the Pod

Update `pod.yml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  namespace: nginx
  labels:
    app: nginx-label
spec:
  containers:
    - name: nginx
      image: nginx:latest
      ports:
        - containerPort: 80

  tolerations:
    - key: "prod"
      operator: "Equal"
      value: "true"
      effect: "NoSchedule"
```

Apply the updated configuration:

```bash
kubectl apply -f pod.yml
```

Check the Pod:

```bash
kubectl get pods -n nginx
```

Kubernetes can now schedule the Pod on a node with the matching taint.

### Step 5: Check which node is running the Pod

```bash
kubectl get pods -n nginx -o wide
```

Example:

```text
NAME        READY   STATUS    NODE
nginx-pod   1/1     Running   minikube-m02
```

The `NODE` column tells you where Kubernetes scheduled the Pod.

In this experiment, the Pod was scheduled on the worker node because it had a matching toleration for that node's taint.

**Remember:** A toleration allows scheduling on a matching node; it does not guarantee that Kubernetes will choose that node.

---

## 6. How to Remove a Taint

To remove a taint, add a hyphen (`-`) at the end of the taint effect.

Syntax:

```bash
kubectl taint nodes <node-name> <key>=<value>:<effect>-
```

Example:

```bash
kubectl taint nodes minikube prod=true:NoSchedule-
```

Verify:

```bash
kubectl describe node minikube
```

The taint should no longer appear.

Once removed, the node no longer has that scheduling restriction.

---

## 7. Useful Commands

| Command | Purpose |
|---|---|
| `kubectl get nodes` | List all nodes. |
| `kubectl describe node minikube` | View node details and taints. |
| `kubectl taint nodes minikube prod=true:NoSchedule` | Apply a taint. |
| `kubectl taint nodes minikube prod=true:NoSchedule-` | Remove a taint. |
| `kubectl get pods -n nginx` | List Pods in the nginx namespace. |
| `kubectl get pods -n nginx -o wide` | Show which node runs each Pod. |
| `kubectl describe pod nginx-pod -n nginx` | Inspect Pod details, scheduling and events. |
| `kubectl apply -f pod.yml` | Create or update resources from YAML. |
| `kubectl delete pod nginx-pod -n nginx` | Delete a Pod. |

---

## 8. Taints vs. Tolerations vs. Node Selectors

These three Kubernetes features serve different purposes.

| Feature | Purpose |
|---|---|
| Taints | Restrict which Pods can run on a node. |
| Tolerations | Allow Pods to run on nodes with matching taints. |
| Node Selectors | Restrict a Pod to nodes with specific labels. |

For example, imagine you have three types of nodes:

- General-purpose nodes
- GPU nodes
- Database nodes

You can use taints to restrict access to GPU nodes and tolerations to allow only selected Pods to use them.

You can then use node selectors or node affinity to guide those Pods toward the GPU nodes.

---

## 9. Common Troubleshooting

### Pod remains Pending

Check the Pod:

```bash
kubectl describe pod nginx-pod -n nginx
```

Look at the Events section.

If you see:

```text
untolerated taint(s)
```

Your Pod may be missing a matching toleration.

### Pod shows ErrImagePull or ImagePullBackOff

This is not necessarily a taint or toleration problem.

It usually means Kubernetes cannot download the container image.

Check the reason:

```bash
kubectl describe pod nginx-pod -n nginx
```

Possible causes include:

- Internet connectivity problems.
- DNS resolution failures.
- Incorrect image names or tags.
- Private image registries requiring authentication.

### Pod is Running but on the wrong node

A toleration does not select a node.

Use `nodeSelector` or node affinity if you want to control which node Kubernetes selects.

---

## 10. Key Takeaways

1. Taints are applied to nodes.
2. Tolerations are added to Pods.
3. `NoSchedule` prevents new Pods without matching tolerations from being scheduled on a tainted node.
4. `PreferNoSchedule` tells Kubernetes to avoid a node when possible.
5. `NoExecute` can evict existing Pods that do not tolerate the taint.
6. A toleration allows a Pod to use a matching node but does not guarantee placement there.
7. Node selectors and node affinity help guide Pods to specific nodes.

**In simple words:** Taints restrict which Pods can enter a node, and tolerations give Pods permission to enter.
