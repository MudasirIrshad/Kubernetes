# Django Notes App — Kubernetes Deployment

This project demonstrates how to deploy a Django Notes App on Kubernetes using Minikube, Docker, and Kubernetes YAML manifests.

The goal is to understand how Kubernetes manages application deployments, creates Pods, and exposes applications through Services.

## Tech Stack

- **Django** — Web application framework
- **Docker** — Containerization
- **Kubernetes** — Container orchestration
- **Minikube** — Local Kubernetes cluster
- **Docker Hub** — Container image registry
- **MySQL** — Database

## Project Architecture

```text
                 Kubernetes Cluster
                         |
                  Namespace: notes-app
                         |
              ┌──────────┴──────────┐
              |                     |
              ▼                     ▼
          Deployment              Service
              |                  ClusterIP
              ▼                     |
          ReplicaSet                |
              |                     |
              ▼                     |
             Pod ◄──────────────────┘
              |
              ▼
       Django Application
           Port 8080
              |
              ▼
           MySQL DB
```

## Project Structure

```text
django-notes-app/
└── k8s/
    ├── namespace.yaml
    ├── deployment.yaml
    └── service.yaml
```

## 1. Namespace

**File:** `namespace.yaml`

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: notes-app
```

A Namespace organizes Kubernetes resources into a logical group.

In this project, the Namespace is named `notes-app`. The Deployment and Service are created inside this Namespace.

Apply the configuration:

```bash
kubectl apply -f namespace.yaml
```

Verify the Namespace:

```bash
kubectl get namespaces
```

## 2. Deployment

**File:** `deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: notes-app-deployment
  labels:
    app: notes-app
  namespace: notes-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: notes-app
  template:
    metadata:
      labels:
        app: notes-app
    spec:
      containers:
        - name: notes-app
          image: mudasirirshad/notes-app-k8s:latest
          ports:
            - containerPort: 8080
          command: ["python", "manage.py", "runserver", "0.0.0.0:8080"]
          env:
            - name: DB_NAME
              value: "test_db"
            - name: DB_USER
              value: "root"
            - name: DB_PASSWORD
              value: "root"
            - name: DB_HOST
              value: "192.168.49.1"
            - name: DB_PORT
              value: "3306"
```

### How it works

- **Deployment:** Declares the desired state of the application and manages updates.
- **Replicas:** Specifies that Kubernetes should maintain one application Pod.
- **Selector:** Identifies Pods managed by the Deployment's ReplicaSet.
- **Template:** Defines the labels, containers, and configuration for Pods created by the Deployment.
- **Image:** Specifies the Docker image used to run the application.
- **Container port:** Documents the port used by Django.
- **Command:** Starts Django's development server on port `8080`.
- **Environment variables:** Configure the application's MySQL connection.

### How Kubernetes creates the Pod

When you apply the Deployment, Kubernetes follows this process:

1. The API server accepts the Deployment configuration.
2. The Deployment controller creates a ReplicaSet.
3. The ReplicaSet creates a Pod using the template.
4. The scheduler assigns the Pod to a suitable node.
5. The node's container runtime obtains the image if necessary and starts the container.
6. Kubernetes monitors the desired state and replaces failed or deleted Pods when necessary.

**Important:** You create a Deployment, not a Pod directly. Kubernetes manages the Pod lifecycle through the ReplicaSet.

## 3. Service

**File:** `service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: notes-app-service
  namespace: notes-app
spec:
  selector:
    app: notes-app
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 8080
  type: ClusterIP
```

### How it works

- **Service:** Provides a stable network endpoint for the application.
- **Selector:** Finds Pods with the label `app: notes-app`.
- **Port:** Exposes port `8080` on the Service.
- **Target port:** Forwards traffic to port `8080` on the selected Pods.
- **ClusterIP:** Makes the Service accessible within the cluster by default.

Pods can be replaced and receive new IP addresses. The Service provides a stable endpoint so clients do not need to track individual Pod IP addresses.

Apply the configuration:

```bash
kubectl apply -f service.yaml
```

## 4. Deploy the Application

Make sure your Kubernetes cluster is running and your Namespace has been created.

From the `k8s` directory, apply the remaining manifests:

```bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
```

Alternatively, apply all three files:

```bash
kubectl apply -f namespace.yaml -f deployment.yaml -f service.yaml
```

Kubernetes will create the resources and work toward the desired state defined in the manifests.

## 5. Verify the Deployment

Check the Deployment:

```bash
kubectl get deployments -n notes-app
```

Check the ReplicaSet:

```bash
kubectl get replicasets -n notes-app
```

Check the Pods:

```bash
kubectl get pods -n notes-app -o wide
```

Check the Service:

```bash
kubectl get services -n notes-app
```

View the resources together:

```bash
kubectl get all -n notes-app
```

Check the Deployment rollout:

```bash
kubectl rollout status deployment/notes-app-deployment -n notes-app
```

## 6. Access the Application

The Service uses `ClusterIP`, so it is not directly exposed outside the cluster.

Use port forwarding to access it locally:

```bash
kubectl port-forward -n notes-app service/notes-app-service 8080:8080
```

Open the following address in your browser:

```text
http://localhost:8080
```

Keep the port-forward command running while accessing the application.

## 7. Troubleshooting

If the Pod does not start, inspect its status:

```bash
kubectl get pods -n notes-app
```

View Pod events and configuration:

```bash
kubectl describe pod -n notes-app -l app=notes-app
```

Check application logs:

```bash
kubectl logs -n notes-app deployment/notes-app-deployment
```

If the Pod is running but the application is inaccessible, verify that Django is listening on port `8080` and that the Service has a ready endpoint.

Check the Service endpoints:

```bash
kubectl get endpoints notes-app-service -n notes-app
```

If the application cannot connect to MySQL, verify database availability, network connectivity, credentials, and MySQL access permissions.

**Database note:** The configured MySQL address, `192.168.49.1`, must be reachable from the Pod. Its suitability depends on your Minikube networking setup.

## Key Kubernetes Concepts Learned

| Concept | Purpose |
|---|---|
| Namespace | Organizes application resources |
| Deployment | Manages the desired application state |
| ReplicaSet | Maintains the desired number of Pods |
| Pod | Provides the environment for running containers |
| Container image | Packages the application and its dependencies |
| Labels | Identify and organize resources |
| Selector | Identifies matching Pods |
| Pod template | Defines how new Pods should be configured |
| Service | Provides a stable network endpoint |
| ClusterIP | Exposes a Service internally within the cluster |
| Port forwarding | Provides temporary local access to the application |

## Important Notes

- The Docker image must exist in a registry accessible to the cluster, or it must be loaded into the cluster's image store.
- The Django application must listen on `0.0.0.0:8080`.
- The MySQL database must be reachable from the Kubernetes Pod.
- The current configuration is intended for learning. Django's development server and hardcoded database credentials should not be used as-is in production.
- For production, use a production-ready Django server, Kubernetes Secrets or an external secrets manager, appropriate network controls, and a suitable database deployment strategy.

---

**Project:** Django Notes App on Kubernetes  
**Purpose:** Learn Kubernetes fundamentals through a practical application deployment.
