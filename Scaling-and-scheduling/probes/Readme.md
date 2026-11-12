# Kubernetes Probes

Kubernetes uses **Probes** to check whether an application inside a container is starting correctly, is healthy, and is ready to receive traffic.

Think of an application as a shop:

- **Startup Probe:** Has the shop finished opening?
- **Liveness Probe:** Is the shop still operating, or is it stuck?
- **Readiness Probe:** Is the shop ready to serve customers?

Kubernetes has three main types of probes.

## 1. Startup Probe

A startup probe checks whether an application has finished starting.

**Analogy:** An old computer may need two minutes to turn on. You should give it time instead of deciding that it is broken immediately.

- Useful for applications that take a long time to start.
- Until the startup probe succeeds, Kubernetes does not run the liveness or readiness probes for that container.
- If startup does not succeed within the configured attempts, Kubernetes restarts the container.

**Remember:** Startup = Has the application finished starting?

## 2. Liveness Probe

A liveness probe checks whether an application is still healthy enough to keep running.

**Analogy:** You call your friend, but they are completely unresponsive. You decide they need to be woken up.

- If the liveness probe fails enough times consecutively, Kubernetes restarts the container.
- Use it to detect situations where an application is stuck and cannot recover by itself.

**Remember:** Liveness = Should Kubernetes restart the container?

## 3. Readiness Probe

A readiness probe checks whether an application is ready to receive requests.

**Analogy:** A restaurant is open, but the chef is still preparing the food. Customers should wait until the restaurant is ready to serve them.

- If readiness fails, Kubernetes marks the Pod as not ready.
- A Pod that is not ready is removed from the ready endpoints used by Kubernetes Services, so Services stop sending normal traffic to it.
- The container is not restarted just because its readiness probe fails.

**Remember:** Readiness = Should the application receive traffic?

## Quick Comparison

| Probe | What does it check? | What happens when it repeatedly fails? |
|---|---|---|
| Startup | Has the application finished starting? | Container is restarted |
| Liveness | Is the application still healthy? | Container is restarted |
| Readiness | Can the application receive requests? | Pod is marked not ready; Service traffic is withheld |

## How Probes Work

Kubernetes supports different ways to perform probe checks:

- **HTTP:** Sends an HTTP request to an endpoint. HTTP status codes from 200 through 399 are considered successful.
- **TCP:** Checks whether a connection can be established to a port.
- **Exec:** Runs a command inside the container. Exit code `0` means success.
- **gRPC:** Checks application health using the gRPC health-checking protocol.

## Practical Example: Nginx Deployment

This example uses HTTP checks against Nginx on port `80`.

Save it as `deployment.yml`:

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

          # Check whether Nginx has finished starting.
          startupProbe:
            httpGet:
              path: /
              port: 80
            periodSeconds: 5
            failureThreshold: 30

          # Restart the container if Nginx becomes unhealthy.
          livenessProbe:
            httpGet:
              path: /
              port: 80
            periodSeconds: 10
            timeoutSeconds: 1
            failureThreshold: 3

          # Send traffic only when Nginx is ready.
          readinessProbe:
            httpGet:
              path: /
              port: 80
            periodSeconds: 10
            timeoutSeconds: 1
            failureThreshold: 3

          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 200m
              memory: 256Mi
```

**Note:** The startup probe is optional for this simple Nginx example. It is included to demonstrate all three probe types. In real applications, choose probe paths and timings that accurately represent application health.

### Apply and Inspect the Deployment

Apply the deployment:

```bash
kubectl apply -f deployment.yml
```

Check Pods:

```bash
kubectl get pods -n nginx
```

Check probe configuration and events:

```bash
kubectl describe pod <pod-name> -n nginx
```

Watch Pod status changes:

```bash
kubectl get pods -n nginx -w
```

Stop watching with `Ctrl + C`.

Check recent namespace events:

```bash
kubectl get events -n nginx --sort-by='.lastTimestamp'
```

## Understanding Pod Status

For a Pod with one container:

| READY | Meaning |
|---|---|
| `1/1` | The one container exists and is ready |
| `0/1` | The container exists but isn't ready |

**Running and Ready are not the same thing.** A container can be running while its readiness probe is failing.

## Practical Experiment: Break the Readiness Check

You can test readiness by changing the HTTP path:

```yaml
readinessProbe:
  httpGet:
    path: /wrong
    port: 80
```

Apply the change:

```bash
kubectl apply -f deployment.yml
```

Watch the Pod:

```bash
kubectl get pods -n nginx -w
```

Nginx should continue running, but the readiness check should fail because `/wrong` normally returns HTTP 404.

Expected result:

- Pod may show `0/1 Running`.
- The container should not restart just because readiness fails.
- Kubernetes Services will stop treating that Pod as a ready destination.

**Important:** Restore the path to `/` after the experiment and apply the file again.

## What I Learned from Troubleshooting

### 1. HTTP 403 or 404 Probe Failures

An HTTP probe expects a successful HTTP response (status code 200–399).

- `403 Forbidden` means the server understood the request but denied access.
- `404 Not Found` means the requested path was not found.

In my Nginx experiment, mounting a PersistentVolumeClaim at `/usr/share/nginx/html` replaced the visible contents of Nginx's default website directory.

If the mounted directory did not contain an accessible index page, Nginx returned `403`.

**Lesson:** When a probe fails, inspect the application, its endpoint, permissions, and mounted volumes. Do not assume the probe configuration itself is always the problem.

### 2. Connection Refused

A message such as:

```text
connect: connection refused
```

means Kubernetes could not establish a connection to the target port at that moment.

The application may not have started listening yet, may have stopped, or may be listening on a different port.

A short-lived failure during startup can be normal. Check whether later probe attempts succeed.

### 3. Liveness Failures Cause Restarts

If liveness fails for the configured number of consecutive attempts, Kubernetes restarts the container.

Readiness failures do not restart the container. They only make the Pod not ready.

### 4. Events Can Show Earlier Failures

`kubectl describe pod` and `kubectl get events` can show historical warnings from startup.

A past warning does not automatically mean the probe is failing now.

Check the current state as well:

```bash
kubectl get pods -n nginx
```

## Important Probe Settings

| Setting | Meaning |
|---|---|
| `initialDelaySeconds` | How long to wait after the container starts before the first probe |
| `periodSeconds` | How often to run the probe |
| `timeoutSeconds` | How long to wait for a probe response |
| `failureThreshold` | How many consecutive failures are needed before the probe is considered failed |
| `successThreshold` | How many consecutive successes are needed to consider the probe successful again |

For startup and liveness probes, `successThreshold` must be `1`. For readiness, it can be higher.

## Remember These Three Lines

- **Startup:** Give the application time to start.
- **Liveness:** Restart it if it gets stuck.
- **Readiness:** Send traffic only when it is ready.

Probes help Kubernetes manage application health and traffic automatically, instead of assuming that a running container is always ready to serve users.
