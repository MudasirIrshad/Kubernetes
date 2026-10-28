# Kubernetes Ingress: A Beginner-Friendly Guide

> **Goal:** Run two applications on Minikube and access both through one hostname:
>
> - NGINX: `http://apps.test/nginx`
> - Django Notes App: `http://apps.test/`
>
> This guide explains the idea from scratch, how to write the YAML, what annotations do, and how to troubleshoot common problems.

---

## 1. What problem does Ingress solve?

Imagine Kubernetes is a large apartment building:

- **Pods** are apartments where your applications live.
- **Services** are reception desks that know which Pods can handle requests.
- **Ingress** is the building's main entrance and signboard. It looks at the address a visitor requests and sends them to the correct reception desk.
- **Ingress Controller** is the security guard who actually reads the Ingress rules and directs visitors.

Without Ingress, you might expose each application separately. With Ingress, one hostname can route different paths to different Services.

For this project:

| Browser URL | Ingress sends the request to | Service port |
|---|---|---:|
| `http://apps.test/nginx` | `nginx-service` | `80` |
| `http://apps.test/` | `notes-app-service` | `8080` |

**Important:** Ingress routes HTTP/HTTPS traffic. It does not run your application, and it does not replace a Deployment or Service.

---

## 2. Understand the traffic flow

When you visit `http://apps.test/nginx`, the request follows this route:

```text
Browser
  |
  | http://apps.test/nginx
  v
Hostname resolves to Minikube IP
  |
  v
NGINX Ingress Controller
  |
  | Reads the Ingress rules:
  | host = apps.test
  | path starts with /nginx
  v
nginx-service:80
  |
  v
NGINX Pod
```

For `http://apps.test/`, the same entrance sends the request to `notes-app-service:8080`, which forwards it to the Django Notes App Pod.

---

## 3. Check your starting environment

These commands assume you are using Linux, Minikube, and the namespace used in this project.

### Check Minikube

```bash
minikube status
minikube ip
```

If Minikube is not running:

```bash
minikube start
```

### Enable the Ingress add-on

```bash
minikube addons enable ingress
```

Check the controller:

```bash
kubectl get pods -n ingress-nginx
```

Wait until the Ingress Controller Pod is `Running` and ready. The controller is the component that watches Ingress resources and configures NGINX to apply their rules.

### Check your application resources

```bash
kubectl get pods -n notes-app-and-nginx-namespace
kubectl get services -n notes-app-and-nginx-namespace
kubectl get endpoints -n notes-app-and-nginx-namespace
```

Replace `notes-app-and-nginx-namespace` if your resources use another namespace.

A Service should have endpoints pointing to ready application Pods. If an endpoint list is empty, check the Service selector and Pod labels.

---

## 4. Make the hostname `apps.test` resolve

A hostname is like a person's name; an IP address is like their street address. Your computer needs to know which IP address belongs to `apps.test`.

First, get the Minikube IP:

```bash
minikube ip
```

For example, if it returns `192.168.49.2`, configure your local hostname resolution so `apps.test` points to that address. One common Linux approach is to add this line to `/etc/hosts`:

```text
192.168.49.2 apps.test
```

Edit the file with:

```bash
sudo nano /etc/hosts
```

Use the actual IP returned by `minikube ip`, not necessarily the example above. Save the file and check resolution:

```bash
getent hosts apps.test
```

The returned address should match your current Minikube IP. If Minikube's IP changes after a restart, update the entry.

> **Note:** Some Minikube setups use a different hostname-resolution method. Use the method configured for your own machine if `/etc/hosts` is not how your environment resolves `apps.test`.

---

## 5. Understand an Ingress YAML file

YAML is a way to describe configuration using indentation. Think of it like an outline: items indented under another item belong to it. Use spaces, not tabs.

A simplified Ingress looks like this:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example-ingress
spec:
  ingressClassName: nginx
  rules:
    - host: apps.test
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: notes-app-service
                port:
                  number: 8080
```

What the main fields mean:

- `apiVersion`: Which Kubernetes API version describes this resource.
- `kind: Ingress`: Tells Kubernetes what kind of resource this is.
- `metadata.name`: The name you use to identify the Ingress.
- `metadata.namespace`: The namespace where the Ingress resource lives. The backend Services must be in the same namespace.
- `spec.ingressClassName: nginx`: Selects the Ingress Controller class named `nginx` in this setup.
- `rules`: The routing instructions.
- `host`: The hostname the rule matches.
- `paths`: The URL paths the rule matches.
- `pathType: Prefix`: Matches the path as a prefix. For example, `/nginx` can match `/nginx` and `/nginx/page`.
- `backend.service.name`: The destination Service name.
- `backend.service.port.number`: The **Service port**, not necessarily the container's port.

---

## 6. Write the Ingress for both applications

The following configuration matches the working setup in this project:

- `/nginx` goes to `nginx-service:80`.
- `/` goes to `notes-app-service:8080`.

Create or open the file:

```bash
mkdir -p ~/Kubernetes/ingress
cd ~/Kubernetes/ingress
nano ingress.yaml
```

Put this YAML in `ingress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: notes-app-ingress
  namespace: notes-app-and-nginx-namespace
spec:
  ingressClassName: nginx
  rules:
    - host: apps.test
      http:
        paths:
          - path: /nginx
            pathType: Prefix
            backend:
              service:
                name: nginx-service
                port:
                  number: 80

          - path: /
            pathType: Prefix
            backend:
              service:
                name: notes-app-service
                port:
                  number: 8080
```

Save in Nano with **Ctrl+O**, press **Enter**, then exit with **Ctrl+X**.

### Why does `/` go to the Notes App?

`/` is the general fallback path for this hostname. The more specific `/nginx` path matches NGINX requests, while other paths match `/` and go to the Notes App.

The Notes App is therefore available at `http://apps.test/` in this working configuration. This is different from placing it at `http://apps.test/app`.

### Check and apply the YAML

```bash
kubectl apply -f ingress.yaml
```

Check the resource:

```bash
kubectl get ingress -n notes-app-and-nginx-namespace
kubectl describe ingress notes-app-ingress -n notes-app-and-nginx-namespace
```

You should see host `apps.test` and the paths mapped to the expected Services.

---

## 7. What is an annotation?

An **annotation** is a key-value setting attached to a Kubernetes resource. It can store extra information or, when a controller recognizes it, change that controller's behavior.

Analogy: Ingress rules are like a delivery address; annotations are like special instructions on the parcel.

For example, the NGINX Ingress Controller understands annotations with names such as:

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
```

This particular annotation tells the NGINX Ingress Controller to rewrite the request path sent to the backend.

### Why might a rewrite be needed?

The standard NGINX welcome page normally responds at `/`. But an Ingress rule that sends `/nginx` to the NGINX Service passes that path along unless a rewrite or backend configuration changes it. The NGINX server may respond with `404 Not Found` because it does not have a page at `/nginx`.

A rewrite can turn a path such as `/nginx` into `/` before it reaches the backend.

### Important warning about rewrite annotations

Do **not** add `nginx.ingress.kubernetes.io/rewrite-target: /` blindly to the combined Ingress above. The annotation applies to the Ingress resource, not just the single path where you wish to use it. It can therefore change requests for the Notes App too, potentially breaking its static files or API routes.

If different paths need different rewrite behavior, use a design that supports those differences—for example, separate Ingress resources with carefully planned, non-conflicting rules, or configure the backends to serve the desired paths. Check all Ingress resources before changing routing:

```bash
kubectl get ingress -A
```

For the working configuration documented here, **no rewrite annotation is required** as long as `/nginx` and `/` are already serving correctly.

### Annotation versus label

- A **label** identifies or groups a resource and can be used by selectors.
- An **annotation** stores additional metadata or controller-specific instructions.

Example label:

```yaml
labels:
  app: nginx
```

Example annotation:

```yaml
annotations:
  nginx.ingress.kubernetes.io/rewrite-target: /
```

A Service selector uses labels to find Pods. An Ingress Controller can use a recognized annotation to change how it handles HTTP requests.

---

## 8. Test the routes

### Test from your browser

Open:

- NGINX: `http://apps.test/nginx`
- Notes App: `http://apps.test/`

### Test with curl

```bash
curl -i http://apps.test/nginx
curl -i http://apps.test/
```

To test routing without relying on local hostname resolution, you can send the `Host` header directly to the Minikube IP:

```bash
curl -i -H 'Host: apps.test' http://$(minikube ip)/nginx
curl -i -H 'Host: apps.test' http://$(minikube ip)/
```

The `Host` header matters because Ingress rules often match the hostname, not only the path.

A `200 OK` generally means the server returned a successful response. A `404` means a server answered but could not find the requested resource. A `502 Bad Gateway` usually means the Ingress Controller could not get a valid response from its upstream Service or Pod.

---

## 9. Common problems and how to troubleshoot them

Troubleshoot from the inside out: first the application Pod, then the Service, then the Ingress, then the hostname/browser. This is like checking a delivery: is the item ready, is the reception desk correct, is the building entrance directing it properly, and is the address correct?

### Problem A: Browser says the site cannot be reached

Check:

```bash
minikube status
minikube ip
getent hosts apps.test
kubectl get pods -n ingress-nginx
```

Possible causes:

- Minikube is stopped.
- `apps.test` does not resolve to the current Minikube IP.
- The Ingress Controller is not ready.
- Port 80 is not reachable in the current setup.

### Problem B: `404 Not Found` at `/nginx`

Check whether the request reaches the correct Service:

```bash
kubectl describe ingress notes-app-ingress -n notes-app-and-nginx-namespace
kubectl get endpoints nginx-service -n notes-app-and-nginx-namespace
```

If the request reaches NGINX but NGINX returns 404, the path may be the issue: the standard welcome page is at `/`, not `/nginx`. Consider a path rewrite or a backend configuration that serves the page at `/nginx`. Do not apply a global rewrite without considering the Notes App paths.

### Problem C: `502 Bad Gateway`

A `502` often means the Ingress Controller cannot connect successfully to the backend.

Check Pods, Services, and endpoints:

```bash
kubectl get pods -n notes-app-and-nginx-namespace
kubectl get services -n notes-app-and-nginx-namespace
kubectl get endpoints -n notes-app-and-nginx-namespace
```

Read the Notes App logs:

```bash
kubectl logs deployment/notes-app-deployment \
  -n notes-app-and-nginx-namespace --tail=100
```

Read the Ingress Controller logs:

```bash
kubectl logs -n ingress-nginx deployment/ingress-nginx-controller --tail=100
```

Common causes include:

- Django failed to start.
- The Service points to the wrong target port.
- The Service selector does not match the Pod labels.
- The Pod is not listening on the expected port.
- The application cannot connect to a required database.

**A Pod being `Running` does not guarantee that the application inside it is functioning.** Read its logs and test the application directly.

### Problem D: Service has no endpoints

Check:

```bash
kubectl describe service notes-app-service -n notes-app-and-nginx-namespace
kubectl get pods -n notes-app-and-nginx-namespace --show-labels
```

The Service selector must match the labels on the application Pods. If it does not, Kubernetes has no Pods to send traffic to.

### Problem E: Wrong Service port

Inspect the Service:

```bash
kubectl get service notes-app-service \
  -n notes-app-and-nginx-namespace -o yaml
```

Ingress should reference the Service's `spec.ports[].port` value, not automatically the container port. For example, a Service can expose port `8080` and forward it to a different `targetPort`.

### Problem F: Django cannot connect to MySQL

In this project, Django uses environment variables such as:

```text
DB_HOST=192.168.49.1
DB_PORT=3306
```

If MySQL is in Docker, confirm the container is running and its port is published:

```bash
docker ps
docker port mysql
```

Test host connectivity:

```bash
nc -vz 127.0.0.1 3306
nc -vz 192.168.49.1 3306
```

Test TCP connectivity from the Django Pod:

```bash
kubectl exec -it deployment/notes-app-deployment \
  -n notes-app-and-nginx-namespace -- \
  python -c "import socket; s=socket.socket(); s.settimeout(3); print(s.connect_ex(('192.168.49.1',3306))); s.close()"
```

A result of `0` means the TCP connection succeeded; it does not prove that the database name, username, password, or permissions are correct. If TCP works but Django still fails, inspect Django logs and database credentials/configuration. Avoid printing passwords when sharing diagnostic output.

### Problem G: React page is blank

The HTML page may load while its JavaScript or CSS files fail. Open the browser's Developer Tools → **Network** and **Console** tabs.

Look for failed requests such as:

```text
/static/js/...
/static/css/...
/api/...
```

Check the failed URL and status code. If the frontend requests root-relative asset paths such as `/static/...`, ensure those paths also reach the correct backend. If the app is moved from `/` to `/app`, its asset base path and API paths may need to be adjusted too.

Do not assume that making `/app` route to Django is enough to make the whole React app work under `/app`.

### Problem H: YAML indentation or validation error

YAML uses indentation to express structure. Use spaces, not tabs, and keep sibling keys aligned.

Validate the file before applying it:

```bash
kubectl apply --dry-run=client -f ingress.yaml
```

Then apply it:

```bash
kubectl apply -f ingress.yaml
```

Inspect the parsed resource:

```bash
kubectl describe ingress notes-app-ingress -n notes-app-and-nginx-namespace
```

### Problem I: Multiple Ingress resources conflict

Check all namespaces:

```bash
kubectl get ingress -A
```

If several Ingress resources use the same hostname and overlapping paths, their behavior can become confusing or controller-specific. Read each resource before deleting anything. Do not delete an Ingress just because its name looks old; first confirm what application depends on it.

---

## 10. Useful command cheat sheet

```bash
# Minikube
minikube status
minikube ip

# Ingress controller
kubectl get pods -n ingress-nginx
kubectl logs -n ingress-nginx deployment/ingress-nginx-controller --tail=100

# Application resources
kubectl get pods -n notes-app-and-nginx-namespace
kubectl get services -n notes-app-and-nginx-namespace
kubectl get endpoints -n notes-app-and-nginx-namespace

# Ingress resources
kubectl get ingress -A
kubectl describe ingress notes-app-ingress -n notes-app-and-nginx-namespace

# Apply and validate
kubectl apply --dry-run=client -f ingress.yaml
kubectl apply -f ingress.yaml

# Test the routes
curl -i http://apps.test/nginx
curl -i http://apps.test/
```

---

## 11. Final mental model

Remember these four jobs:

1. **Deployment** keeps your application Pods running.
2. **Service** gives those Pods a stable internal destination.
3. **Ingress rules** match a hostname and path and choose a Service.
4. **Ingress Controller** reads the rules and actually handles incoming HTTP/HTTPS requests.

Annotations are optional extra instructions for a controller. Use them when you need behavior such as path rewriting, but understand their scope before applying them.

For this project, the target working routes are:

```text
http://apps.test/nginx  -> nginx-service:80
http://apps.test/       -> notes-app-service:8080
```

If something breaks, diagnose the route layer by layer rather than changing several YAML files at once.
