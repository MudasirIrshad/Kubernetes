# Kubernetes Fundamentals — Beginner Guide

This repository contains practical Kubernetes examples for learners.

We will use **NGINX** throughout this session so that each Kubernetes concept can be understood by creating, inspecting, scaling, and updating real resources.

---

# Table of Contents

1. [What is Kubernetes?](#1-what-is-kubernetes)
2. [Namespaces](#2-namespaces)
3. [Default Kubernetes Namespaces](#3-default-kubernetes-namespaces)
4. [Namespace Analogy](#4-namespace-analogy)
5. [Creating a Namespace](#5-creating-a-namespace)
6. [YAML / Manifest Files](#6-yaml--manifest-files)
7. [Creating a Pod](#7-creating-a-pod)
8. [Pod and Container](#8-pod-and-container)
9. [Inspecting a Pod](#9-inspecting-a-pod)
10. [Entering a Pod](#10-entering-a-pod)
11. [Pod Events](#11-pod-events)
12. [Pod vs Deployment](#12-pod-vs-deployment)
13. [Labels](#13-labels)
14. [Selectors](#14-selectors)
15. [Labels and Selectors Together](#15-labels-and-selectors-together)
16. [Deployment](#16-deployment)
17. [Deployment Structure](#17-deployment-structure)
18. [Deployment → ReplicaSet → Pods](#18-deployment--replicaset--pods)
19. [ReplicaSet](#19-replicaset)
20. [Deployment vs ReplicaSet](#20-deployment-vs-replicaset)
21. [ReplicationController](#21-replicationcontroller)
22. [StatefulSet](#22-statefulset)
23. [Scaling a Deployment](#23-scaling-a-deployment)
24. [Rolling Updates](#24-rolling-updates)
25. [Checking the Current Image](#25-checking-the-current-image)
26. [Rollout History](#26-rollout-history)
27. [Annotations](#27-annotations)
28. [Useful Kubernetes Commands](#28-useful-kubernetes-commands)
29. [Complete Resource Relationship](#29-complete-resource-relationship)
30. [Key Concepts to Remember](#30-key-concepts-to-remember)
31. [Practical Learning Exercise](#31-practical-learning-exercise)

---

# 1. What is Kubernetes?

Kubernetes is a platform for managing containerized applications.

Instead of manually running containers and managing them ourselves, Kubernetes can:

- Run containers
- Restart failed containers
- Create multiple copies of an application
- Scale applications
- Perform rolling updates
- Roll back previous versions
- Organize resources
- Manage networking and services

A simple mental model:

```text
Kubernetes Cluster
│
├── Namespace
│
├── Deployment
│      │
│      └── ReplicaSet
│             │
│             ├── Pod
│             └── Pod
│
├── Service
├── ConfigMap
└── Secret
```

---

# 2. Namespaces

A **Namespace** is a logical space inside a Kubernetes cluster.

It helps organize Kubernetes resources.

For example:

```text
Kubernetes Cluster
│
├── nginx
│   ├── Pods
│   ├── Deployments
│   └── Services
│
├── development
│   ├── Pods
│   └── Deployments
│
└── production
    ├── Pods
    └── Deployments
```

Namespaces are useful when a cluster contains resources for different applications, teams, or environments.

---

# 3. Default Kubernetes Namespaces

A Kubernetes cluster normally contains several namespaces.

Check them with:

```bash
kubectl get namespaces
```

Or using the `k` alias:

```bash
k get ns
```

## `default`

The normal/default workspace.

If you don't specify a namespace when creating a resource, Kubernetes normally creates it in:

```text
default
```

For example:

```bash
k run nginx --image=nginx
```

The Pod will be created in the `default` namespace.

To create it in another namespace:

```bash
k run nginx --image=nginx -n nginx
```

---

## `kube-system`

Contains Kubernetes system components.

---

## `kube-public`

Contains information that can be publicly/readably accessed within the cluster.

---

## `kube-node-lease`

Used for node heartbeat/lease information.

---

# 4. Namespace Analogy

Think of a Kubernetes Namespace like a **WhatsApp group**.

A WhatsApp group has:

- Members
- Admins
- Messages
- Group settings
- Group rules
- Its own space

A Kubernetes Namespace also provides a logical space for resources.

For example:

```text
WhatsApp
│
├── Family Group
├── Friends Group
└── Work Group
```

Kubernetes:

```text
Cluster
│
├── nginx namespace
├── development namespace
└── production namespace
```

This is an analogy to understand the concept. A Namespace is not literally a WhatsApp group.

Namespace reference diagram:

https://cdn.hashnode.com/res/hashnode/image/upload/v1682743099019/bc53ef33-25d5-4c54-ba3d-ad6fcb30848c.png

---

# 5. Creating a Namespace

## Using the command line

```bash
k create namespace nginx
```

Check it:

```bash
k get namespaces
```

Or:

```bash
k get ns
```

---

## Using YAML

Create:

```text
namespace.yml
```

Content:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: nginx
```

Apply it:

```bash
k apply -f namespace.yml
```

Check:

```bash
k get ns
```

Delete it:

```bash
k delete namespace nginx
```

Or:

```bash
k delete ns nginx
```

You can recreate it from the YAML:

```bash
k apply -f namespace.yml
```

---

# 6. YAML / Manifest Files

There are two common ways to create Kubernetes resources.

## Using a command

```bash
k create namespace nginx
```

This is useful for:

- Learning
- Quick testing
- One-time tasks

But the command itself is not stored as a configuration file.

---

## Using a manifest

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: nginx
```

Then:

```bash
k apply -f namespace.yml
```

The manifest can be:

- Saved
- Reused
- Committed to Git
- Pushed to GitHub
- Shared with other developers
- Used to recreate the resource

### Simple analogy

```text
CLI command
    ↓
"Do this now"

Manifest
    ↓
"This is what I want Kubernetes to have"
```

This is why YAML manifests are heavily used in real Kubernetes projects.

---

# 7. Creating a Pod

A Pod is the smallest deployable unit in Kubernetes.

A Pod runs one or more containers.

For our example, we will run NGINX.

## Using `kubectl run`

```bash
k run nginx-pod --image=nginx -n nginx
```

Check it:

```bash
k get pods -n nginx
```

Example:

```text
NAME         READY   STATUS    RESTARTS   AGE
nginx-pod    1/1     Running   0          10s
```

---

# 8. Pod and Container

A common beginner question is:

> "When we use `k run`, where is the container?"

For example:

```bash
k run nginx-pod --image=nginx -n nginx
```

You don't explicitly write:

```yaml
containers:
```

because `kubectl run` generates the Pod configuration for you.

The resulting Pod still contains a container.

Conceptually:

```text
Pod
│
└── Container
      │
      └── NGINX
```

With YAML, you explicitly define the container:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  namespace: nginx

spec:
  containers:
    - name: nginx
      image: nginx:latest
      ports:
        - containerPort: 80
```

Notice:

```yaml
spec:
  containers:
```

A Pod always contains one or more containers.

---

# 9. Inspecting a Pod

To get a simple overview:

```bash
k get pods -n nginx
```

For detailed information:

```bash
k describe pod nginx-pod -n nginx
```

`describe` can show:

- Pod status
- Container information
- Image
- IP address
- Node
- Ports
- Restart count
- Conditions
- Volumes
- Events

Example:

```text
Name:        nginx-pod
Namespace:   nginx
Status:      Running

Containers:
  nginx:
    Image:    nginx:latest
```

---

# 10. Entering a Pod

If you want to work inside a running container:

```bash
k exec -it pod/nginx-pod -n nginx -- bash
```

Breakdown:

```text
k exec
    ↓
Execute a command inside a Pod

-it
    ↓
Interactive terminal

pod/nginx-pod
    ↓
The Pod we want to enter

-n nginx
    ↓
Namespace

-- bash
    ↓
Open Bash shell
```

If the container does not have Bash, try:

```bash
k exec -it pod/nginx-pod -n nginx -- sh
```

---

# 11. Pod Events

You can see Pod events using:

```bash
k describe pod nginx-pod -n nginx
```

At the bottom you may see:

```text
Events:

Normal  Scheduled
Normal  Pulling
Normal  Pulled
Normal  Created
Normal  Started
```

A typical startup sequence is:

```text
Scheduled
    ↓
Pulling
    ↓
Pulled
    ↓
Created
    ↓
Started
```

These events are extremely useful for troubleshooting.

For example, you might see:

```text
FailedScheduling
ImagePullBackOff
ErrImagePull
CrashLoopBackOff
```

when something goes wrong.

---

# 12. Pod vs Deployment

A Pod can run your application.

But if you want your application to be:

- Scalable
- Highly available
- Updated safely
- Rolled back
- Automatically recreated

you normally use a **Deployment**.

Simple mental model:

```text
Pod
↓
Runs the application
```

```text
Deployment
↓
Manages application Pods
```

A useful analogy:

> Creating one Pod is not a big thing. Managing many Pods and keeping the application running is where Deployment becomes useful.

---

# 13. Labels

A **Label** is a key-value tag attached to a Kubernetes resource.

Example:

```yaml
metadata:
  labels:
    app: nginx
```

This means:

```text
Label:
app = nginx
```

Another example:

```yaml
labels:
  app: nginx
  environment: production
```

Labels help Kubernetes identify and group resources.

---

## Check Pod labels

```bash
k get pods -n nginx --show-labels
```

Example:

```text
NAME        LABELS
nginx-pod   app=nginx
```

You can add a label using:

```bash
k label pod nginx-pod app=nginx-label -n nginx
```

Check it:

```bash
k get pod nginx-pod -n nginx --show-labels
```

---

# 14. Selectors

A **Selector** finds resources based on their labels.

Simple definition:

```text
Label
    ↓
Tags the resource

Selector
    ↓
Finds the resource using the tag
```

Example label:

```yaml
labels:
  app: nginx
```

Selector:

```yaml
selector:
  matchLabels:
    app: nginx
```

The selector says:

> Find Pods whose label is `app=nginx`.

---

# 15. Labels and Selectors Together

Suppose a Pod has:

```yaml
labels:
  app: nginx
```

A selector:

```yaml
selector:
  matchLabels:
    app: nginx
```

matches that Pod.

You can also use:

```yaml
labels:
  environment: production
```

with:

```yaml
matchLabels:
  environment: production
```

`matchLabels` requires an exact label match.

Remember:

```text
Label = identifies/tags a resource

Selector = finds resources using labels
```

---

# 16. Deployment

A Deployment manages application Pods.

Example:

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
          image: nginx:latest
          ports:
            - containerPort: 80
```

Apply it:

```bash
k apply -f deployment.yml
```

Check it:

```bash
k get deployment -n nginx
```

---

# 17. Deployment Structure

The most important part of a Deployment is understanding its structure.

```yaml
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
          image: nginx:latest
```

There are two different `spec` sections here.

## Deployment spec

```yaml
spec:
  replicas: 2
  selector:
```

This describes how the Deployment manages the application.

## Pod template spec

```yaml
template:
  metadata:
    labels:
  spec:
    containers:
```

This describes what the Pods created by the Deployment should look like.

---

# 18. Deployment → ReplicaSet → Pods

A Deployment creates and manages ReplicaSets, while ReplicaSets manage Pods.

The normal relationship is:

```text
Deployment
     ↓
ReplicaSet
     ↓
Pods
     ↓
Containers
     ↓
Application
```

For NGINX:

```text
nginx-deployment
        ↓
nginx-deployment-65bb84cffc
        ↓
┌───────────────────────┐
│                       │
Pod                     Pod
│                       │
NGINX                   NGINX
```

Check the Deployment:

```bash
k get deployment -n nginx
```

Check ReplicaSets:

```bash
k get rs -n nginx
```

Check Pods:

```bash
k get pods -n nginx
```

---

# 19. ReplicaSet

A ReplicaSet's main responsibility is to maintain the desired number of Pods.

For example:

```yaml
apiVersion: apps/v1
kind: ReplicaSet

metadata:
  name: nginx-rs
  namespace: nginx

spec:
  replicas: 3

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
          image: nginx:1.27
```

This means:

> Keep 3 matching Pods running.

If one Pod dies:

```text
3 Pods
  ↓
1 Pod dies
  ↓
2 Pods
  ↓
ReplicaSet creates another
  ↓
3 Pods
```

ReplicaSet maintains the desired count.

---

# 20. Deployment vs ReplicaSet

Both can maintain multiple Pods.

The important difference is their responsibility.

## ReplicaSet

Main purpose:

> Keep the desired number of Pods running.

```text
ReplicaSet
   │
   ├── Pod
   ├── Pod
   └── Pod
```

## Deployment

Main purpose:

> Manage the application lifecycle, including ReplicaSets, rolling updates, and rollbacks.

```text
Deployment
    │
    ├── Old ReplicaSet
    │
    └── New ReplicaSet
            │
            ├── Pod
            └── Pod
```

### Simple comparison

| Resource | Main responsibility |
|---|---|
| Pod | Runs containers |
| ReplicaSet | Maintains desired number of Pods |
| Deployment | Manages ReplicaSets, updates, rollbacks, and scaling |

In modern Kubernetes applications, **Deployment is normally used instead of directly creating a ReplicaSet**.

The Deployment creates and manages ReplicaSets for you.

---

# 21. ReplicationController

ReplicationController is an older Kubernetes resource.

Its purpose was similar to ReplicaSet:

```text
ReplicationController
        ↓
      Pods
```

The general progression was:

```text
ReplicationController
        ↓
     ReplicaSet
        ↓
     Deployment
```

ReplicaSet is the modern replacement for ReplicationController.

In modern Kubernetes applications, you normally use:

```yaml
kind: Deployment
```

rather than manually creating a ReplicationController.

---

# 22. StatefulSet

Deployment is commonly used for **stateless applications**.

For example:

```text
NGINX
API server
Web application
```

Stateful applications often require stable identity and storage.

Examples include:

```text
Databases
Distributed systems
Stateful services
```

For these use cases, Kubernetes provides:

```yaml
kind: StatefulSet
```

A simplified relationship:

```text
Deployment
    ↓
Stateless application
```

```text
StatefulSet
    ↓
Stateful application
```

StatefulSet provides features such as stable Pod identity, stable storage associations, and ordered operations.

---

# 23. Scaling a Deployment

Suppose your Deployment currently has:

```yaml
replicas: 2
```

You can scale it directly from the command line.

```bash
k scale deployment/nginx-deployment -n nginx --replicas=1
```

This changes the desired number of Deployment-managed Pods from 2 to 1.

To scale to 5:

```bash
k scale deployment/nginx-deployment -n nginx --replicas=5
```

Check:

```bash
k get pods -n nginx
```

And:

```bash
k get deployment nginx-deployment -n nginx
```

---

## Manual scaling vs automatic scaling

`kubectl scale` is a manual scaling action.

```text
You
 ↓
kubectl scale
 ↓
Change replicas
```

Kubernetes also supports automatic scaling using HPA:

```text
HPA
 ↓
Monitors metrics
 ↓
Adjusts replicas
```

So:

```text
kubectl scale = manual scaling

HPA = automatic scaling
```

---

# 24. Rolling Updates

One of the biggest advantages of Deployment is rolling updates.

Suppose your Deployment currently uses:

```yaml
image: nginx:latest
```

You want to change it to:

```text
nginx:1.27
```

You can update the Deployment using:

```bash
k set image deployment/nginx-deployment nginx=nginx:1.27 -n nginx
```

Kubernetes creates a new ReplicaSet and gradually replaces the old Pods.

Conceptually:

```text
Old version
nginx:latest
     ↓
Old ReplicaSet
     ↓
Old Pods

        ↓
    Rolling Update
        ↓

New ReplicaSet
     ↓
New Pods
nginx:1.27
```

This avoids replacing everything at once.

---

# 25. Checking the Current Image

Before changing an image, check what your Pods are currently running.

Use:

```bash
k get pods -n nginx -o custom-columns="NAME:.metadata.name,IMAGE:.spec.containers[*].image"
```

Example:

```text
NAME                               IMAGE
nginx-deployment-7558b6b8b-cmjbj   nginx:latest
nginx-pod                          nginx:latest
```

You can also check the Deployment's configured image:

```bash
k get deployment nginx-deployment -n nginx -o jsonpath='{.spec.template.spec.containers[*].image}{"\n"}'
```

This is particularly important because a manually created Pod and a Deployment-managed Pod can have different ownership.

---

# 26. Rollout History

After changing an image, you can check Deployment rollout history:

```bash
k rollout history deployment/nginx-deployment -n nginx
```

Check the rollout status:

```bash
k rollout status deployment/nginx-deployment -n nginx
```

A successful rollout shows:

```text
deployment "nginx-deployment" successfully rolled out
```

---

## Rollback

If a Deployment update causes a problem, you can roll back:

```bash
k rollout undo deployment/nginx-deployment -n nginx
```

Then check:

```bash
k rollout status deployment/nginx-deployment -n nginx
```

This is one of the major reasons Deployment is preferred over directly managing ReplicaSets.

---

# 27. Annotations

Annotations are additional metadata attached to Kubernetes resources.

Think of them as **notes/information** about a resource.

### Label

```yaml
labels:
  app: nginx
```

A label is a tag that Kubernetes can use for selection.

### Annotation

```yaml
annotations:
  description: "My NGINX learning Pod"
```

An annotation stores additional information.

Example:

```yaml
metadata:
  name: nginx-pod

  labels:
    app: nginx

  annotations:
    description: "My NGINX learning Pod"
    owner: "DevOps team"
```

You can add an annotation using:

```bash
k annotate pod nginx-pod -n nginx description="My NGINX learning pod"
```

Check it:

```bash
k describe pod nginx-pod -n nginx
```

You will see:

```text
Annotations:
  description: My NGINX learning pod
```

### Label vs Annotation

| | Label | Annotation |
|---|---|---|
| Purpose | Identify/tag resources | Store additional information |
| Used by selectors | Yes | No |
| Used for grouping | Yes | Normally no |
| Stores metadata | Yes | Yes |
| Example | `app: nginx` | `description: "NGINX server"` |

Remember:

```text
Label      = Tag

Annotation = Note
```

---

# 28. Useful Kubernetes Commands

## Get namespaces

```bash
k get ns
```

## Get Pods

```bash
k get pods
```

## Get Pods from a namespace

```bash
k get pods -n nginx
```

## Get Pods with labels

```bash
k get pods -n nginx --show-labels
```

## Get Deployments

```bash
k get deployments -n nginx
```

## Get ReplicaSets

```bash
k get rs -n nginx
```

## Describe a Pod

```bash
k describe pod nginx-pod -n nginx
```

## Describe a Deployment

```bash
k describe deployment nginx-deployment -n nginx
```

## Execute a command inside a Pod

```bash
k exec -it pod/nginx-pod -n nginx -- bash
```

## Apply a manifest

```bash
k apply -f deployment.yml
```

## Delete a resource

```bash
k delete pod nginx-pod -n nginx
```

## Scale a Deployment

```bash
k scale deployment/nginx-deployment -n nginx --replicas=3
```

## Change an image

```bash
k set image deployment/nginx-deployment nginx=nginx:1.27 -n nginx
```

## Check rollout status

```bash
k rollout status deployment/nginx-deployment -n nginx
```

## Check rollout history

```bash
k rollout history deployment/nginx-deployment -n nginx
```

## Roll back

```bash
k rollout undo deployment/nginx-deployment -n nginx
```

## Check Pod image

```bash
k get pods -n nginx -o custom-columns="NAME:.metadata.name,IMAGE:.spec.containers[*].image"
```

---

# 29. Complete Resource Relationship

The most important relationship from this session is:

```text
Kubernetes Cluster
│
├── Namespace
│
└── Deployment
      │
      ↓
   ReplicaSet
      │
      ↓
     Pods
      │
      ↓
  Containers
      │
      ↓
  Application
```

For our NGINX example:

```text
nginx Namespace
│
└── nginx-deployment
        │
        ↓
nginx-deployment-7558b6b8b
        │
        ├── nginx-deployment-7558b6b8b-cmjbj
        │       │
        │       └── nginx:1.27
        │
        └── other Pods when replicas > 1
```

---

# 30. Key Concepts to Remember

## Namespace

```text
Logical space for Kubernetes resources
```

Analogy:

```text
Namespace ≈ WhatsApp Group
```

---

## Pod

```text
Smallest deployable unit in Kubernetes
```

A Pod contains one or more containers.

```text
Pod
└── Container
```

---

## Container

Runs the actual application.

Example:

```text
NGINX
```

---

## Label

```text
Tag attached to a resource
```

Example:

```yaml
app: nginx
```

---

## Selector

```text
Finds resources using labels
```

Example:

```yaml
matchLabels:
  app: nginx
```

---

## ReplicaSet

```text
Maintains the desired number of matching Pods
```

Example:

```text
replicas: 3
```

means:

```text
Keep 3 Pods running
```

---

## Deployment

```text
Manages ReplicaSets and application lifecycle
```

Provides:

- Scaling
- Rolling updates
- Rollbacks
- Revision history

---

## StatefulSet

Used for applications that require stable identity and storage.

Common examples:

```text
Databases
Distributed systems
Stateful applications
```

---

## Annotation

```text
Additional information/metadata attached to a resource
```

Remember:

```text
Label      = Tag

Annotation = Note
```

---

# 31. Practical Learning Exercise

After completing this session, try the following yourself.

## 1. Create a namespace

```bash
k create ns practice
```

## 2. Create an NGINX Pod

```bash
k run nginx-pod --image=nginx:latest -n practice
```

## 3. Check the Pod

```bash
k get pods -n practice
```

## 4. Add a label

```bash
k label pod nginx-pod app=nginx -n practice
```

## 5. Check the label

```bash
k get pods -n practice --show-labels
```

## 6. Create a Deployment

Create a `deployment.yml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: practice

spec:
  replicas: 2

  selector:
    matchLabels:
      app: nginx-deployment

  template:
    metadata:
      labels:
        app: nginx-deployment

    spec:
      containers:
        - name: nginx
          image: nginx:latest
          ports:
            - containerPort: 80
```

Apply it:

```bash
k apply -f deployment.yml
```

## 7. Check the Deployment

```bash
k get deployment -n practice
```

## 8. Check the ReplicaSet

```bash
k get rs -n practice
```

## 9. Check the Pods

```bash
k get pods -n practice --show-labels
```

## 10. Scale the Deployment

```bash
k scale deployment/nginx-deployment -n practice --replicas=5
```

Check:

```bash
k get pods -n practice
```

## 11. Change the NGINX version

```bash
k set image deployment/nginx-deployment nginx=nginx:1.27 -n practice
```

Watch the rollout:

```bash
k rollout status deployment/nginx-deployment -n practice
```

## 12. Check the image

```bash
k get pods -n practice -o custom-columns="NAME:.metadata.name,IMAGE:.spec.containers[*].image"
```

## 13. Check rollout history

```bash
k rollout history deployment/nginx-deployment -n practice
```

## 14. Practice rollback

```bash
k rollout undo deployment/nginx-deployment -n practice
```

Then:

```bash
k rollout status deployment/nginx-deployment -n practice
```

---

# Final Mental Model

If you remember only one diagram from this session, remember this:

```text
                    Kubernetes
                        │
                        ↓
                   Namespace
                        │
                        ↓
                   Deployment
                        │
                        ↓
                   ReplicaSet
                        │
              ┌─────────┴─────────┐
              ↓                   ↓
             Pod                 Pod
              │                   │
              ↓                   ↓
          Container           Container
              │                   │
              ↓                   ↓
           NGINX                NGINX
```

And remember the responsibilities:

```text
Pod
↓
Runs the application


ReplicaSet
↓
Keeps the desired number of Pods alive


Deployment
↓
Manages ReplicaSets
↓
Handles scaling
↓
Handles rolling updates
↓
Handles rollbacks


Namespace
↓
Organizes resources


Label
↓
Tags resources


Selector
↓
Finds resources using labels


Annotation
↓
Stores additional information
```

For normal stateless applications, the common Kubernetes pattern is:

```text
Deployment
    ↓
ReplicaSet
    ↓
Pods
    ↓
Containers
```

You normally create the **Deployment** and let Kubernetes create and manage the ReplicaSet and Pods for you.

---

# Most Important Takeaways

```text
Namespace
    → Organizes resources

Pod
    → Runs containers

Container
    → Runs the actual application

Label
    → Tags a resource

Selector
    → Finds resources using labels

ReplicaSet
    → Maintains the desired number of Pods

Deployment
    → Manages ReplicaSets, scaling, updates and rollbacks

StatefulSet
    → Manages stateful applications

Annotation
    → Stores additional information
```

> **Learn the relationship, not just the commands:**
>
> **Namespace → Deployment → ReplicaSet → Pod → Container → Application**
