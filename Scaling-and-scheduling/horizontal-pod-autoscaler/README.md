# Kubernetes Horizontal Pod Autoscaler (HPA) --- Hands-on Practice

This folder contains two small projects that help you understand
**Horizontal Pod Autoscaling (HPA)** in Kubernetes:

1.  **Apache project** --- a simple web server.
2.  **Django Notes App project** --- a small application that uses a
    database.

Both projects teach the same main idea: Kubernetes can automatically
increase or decrease the number of running Pods when CPU usage changes.

The guide uses simple English, small examples, and copy-and-paste
commands. You do not need to understand everything on the first read.
Follow the steps in order and watch what Kubernetes does.

------------------------------------------------------------------------

## 1. What will you learn?

After completing these projects, you should understand:

-   What a Pod, Deployment, and Service do.
-   What HPA is and why we use it.
-   The difference between scaling manually and scaling automatically.
-   Why CPU requests are important for CPU-based HPA.
-   What Metrics Server does.
-   How to create artificial traffic to test autoscaling.
-   How to watch HPA increase and decrease the number of Pods.
-   Why HPA scales the application Pods, not the database.

## 2. The basic idea: imagine a restaurant

Imagine you own a restaurant.

-   **Pod:** One worker who serves customers.
-   **Deployment:** The manager who makes sure the correct number of
    workers are available.
-   **Service:** The front desk that sends each customer to an available
    worker.
-   **HPA:** The manager's assistant who watches how busy the workers
    are and asks the manager to add or remove workers.
-   **Metrics Server:** The person who reports how busy each worker is.

If only a few customers arrive, one worker may be enough.

If many customers arrive and the workers become busy, HPA can ask
Kubernetes to start more workers.

When customers leave and the workers become less busy, HPA can reduce
the number of workers again.

In Kubernetes, these workers are **Pods**.

------------------------------------------------------------------------

## 3. Important Kubernetes concepts

### 3.1 Pod

A Pod is the smallest unit Kubernetes runs.

Think of a Pod as one running copy of your application.

For example, if your Django application has three Pods, Kubernetes is
running three copies of that application.

``` bash
kubectl get pods -n notes-app
```

The command lists the Pods in the `notes-app` namespace.

### 3.2 Deployment

A Deployment manages your Pods.

You normally do not create and manage application Pods one by one.
Instead, you create a Deployment and tell it how many replicas you want.

Example:

``` yaml
spec:
  replicas: 2
```

This tells Kubernetes:

> Keep two copies of this application running.

If a Pod crashes, the Deployment tries to create another one to maintain
the requested number.

### 3.3 Replica

A replica means one copy of an application Pod.

For example:

-   `replicas: 1` means one Pod.
-   `replicas: 3` means three Pods.
-   `replicas: 5` means five Pods.

### 3.4 Service

A Service gives your application a stable network address.

Imagine you have three workers. Customers should not need to remember
the name of each worker. They can go to one reception desk, and the desk
sends them to a worker.

A Kubernetes Service works in a similar way. It gives clients one stable
address and sends traffic to matching Pods.

In this project, the Apache Service is:

``` text
apache-service
```

The Notes App Service is:

``` text
notes-app-service
```

### 3.5 Namespace

A namespace is a way to separate Kubernetes resources into groups.

Think of namespaces as separate rooms in one building.

Our projects use these namespaces:

  Project            Namespace
  ------------------ -------------
  Apache             `apache-ns`
  Django Notes App   `notes-app`

Always use the correct namespace when running commands.

------------------------------------------------------------------------

## 4. What is HPA?

**HPA means Horizontal Pod Autoscaler.**

It automatically changes the number of Pod replicas in a Deployment,
StatefulSet, or another supported scalable resource.

The word **horizontal** means adding or removing more copies.

For example:

``` text
Low traffic:       1 Pod
More traffic:      2 Pods
High traffic:      4 Pods
Traffic decreases: 2 Pods
Very low traffic:  1 Pod
```

HPA does not make one Pod bigger. It changes how many Pods are running.

### Horizontal scaling vs. vertical scaling

Imagine one worker is becoming too busy.

-   **Horizontal scaling:** Hire more workers.
-   **Vertical scaling:** Give the same worker better equipment or more
    resources.

In Kubernetes:

  -----------------------------------------------------------------------
  Type                    What changes?           Example
  ----------------------- ----------------------- -----------------------
  HPA                     Number of Pods          1 Pod becomes 4 Pods

  VPA                     CPU/memory requests for A Pod gets a higher
                          a Pod                   memory request

  Cluster Autoscaler      Number of cluster nodes A cluster gets another
                                                  worker node
  -----------------------------------------------------------------------

These tools solve different problems. HPA does not automatically add a
new machine to your cluster. It adds Pods. If the cluster does not have
enough capacity, the new Pods may remain Pending until capacity becomes
available.

------------------------------------------------------------------------

## 5. How HPA works

For our CPU-based examples, the process looks like this:

``` text
Application Pods use CPU
          |
          v
Metrics Server collects CPU metrics
          |
          v
HPA reads the metrics
          |
          v
HPA compares actual CPU utilization with the target
          |
          v
HPA asks the Deployment to change the replica count
          |
          v
Deployment creates or removes Pods
```

### What is Metrics Server?

Metrics Server collects resource usage information from Kubernetes nodes
and Pods.

HPA needs metrics to make CPU- or memory-based scaling decisions.

In Minikube, you can enable it with:

``` bash
minikube addons enable metrics-server
```

Check whether it is running:

``` bash
kubectl get pods -n kube-system
```

Look for a Pod whose name starts with:

``` text
metrics-server
```

You can also check resource metrics:

``` bash
kubectl top nodes
kubectl top pods -n notes-app
```

If `kubectl top` reports that metrics are unavailable, wait a little and
check that Metrics Server is running.

**Important:** Metrics Server reports resource usage. It does not
generate traffic and it does not decide how many Pods should run. HPA
makes the scaling decision.

------------------------------------------------------------------------

## 6. CPU requests, limits, and HPA

This is one of the most important parts of CPU-based HPA.

Consider this container configuration:

``` yaml
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 200m
    memory: 256Mi
```

Let's understand it.

### CPU request

``` yaml
requests:
  cpu: 100m
```

A CPU request tells Kubernetes how much CPU the container is expected to
need for scheduling.

`100m` means 100 millicores, or one tenth of one CPU core.

For CPU-based HPA using `averageUtilization`, the request is also the
reference value used to calculate utilization.

### CPU limit

``` yaml
limits:
  cpu: 200m
```

A CPU limit sets the maximum CPU amount the container can use.

It is not the value used as the denominator for HPA's CPU utilization
percentage.

### Simple percentage example

Suppose a Pod has a CPU request of `100m`.

If it uses `20m` CPU:

``` text
20m usage / 100m request = 20%
```

If it uses `50m` CPU:

``` text
50m usage / 100m request = 50%
```

If it uses `100m` CPU:

``` text
100m usage / 100m request = 100%
```

So, if your HPA target is 20%, HPA compares the measured average CPU
utilization against 20% of the CPU requests.

**Important:** A target of `20` means 20% of the CPU request. It does
not mean 20 CPU cores, 20 millicores, or 20% of the CPU limit.

### Why does HPA need CPU requests?

If a container has no CPU request, Kubernetes does not have the normal
request value needed to calculate CPU utilization for this type of HPA
metric. The CPU utilization metric may show as unavailable, and HPA may
not be able to make the expected scaling decision.

For CPU utilization-based HPA, define a CPU request on the application
container.

------------------------------------------------------------------------

# Project 1: Apache Web Server

## 7. What are we building?

In this project, we run an Apache web server inside Kubernetes.

We create:

-   A namespace named `apache-ns`.
-   An Apache Deployment.
-   An Apache Service.
-   An HPA that can scale the Deployment from 1 to 5 Pods.
-   A BusyBox load-generator Pod that sends repeated HTTP requests.

The goal is to create traffic and watch HPA add more Apache Pods. Then
we stop the traffic and watch HPA reduce the number of Pods.

### How the project works

``` text
BusyBox load-generator
          |
          | sends repeated HTTP requests
          v
   apache-service
          |
          | sends requests to matching Pods
          v
   Apache Pod 1
   Apache Pod 2
   Apache Pod 3
   ...
          |
          v
   HPA watches CPU usage
```

The Service is the stable address. The Deployment manages Apache Pods.
HPA changes the Deployment's replica count.

## 8. Create the Apache namespace

``` bash
kubectl create namespace apache-ns
```

Check it:

``` bash
kubectl get namespaces
```

If the namespace already exists, you do not need to create it again.

## 9. Apache Deployment

Create a file named `deployment.yaml` in your Apache project folder:

``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: apache-deployment
  namespace: apache-ns
spec:
  replicas: 1
  selector:
    matchLabels:
      app: apache
  template:
    metadata:
      labels:
        app: apache
    spec:
      containers:
        - name: apache
          image: httpd:latest
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 200m
              memory: 256Mi
```

Apply it:

``` bash
kubectl apply -f deployment.yaml
```

Check the Deployment and Pods:

``` bash
kubectl get deployments -n apache-ns
kubectl get pods -n apache-ns
```

### Understand the important parts

  ----------------------------------------------------------------------------
  YAML field                               Simple meaning
  ---------------------------------------- -----------------------------------
  `kind: Deployment`                       Create a Deployment

  `replicas: 1`                            Start with one Pod

  `matchLabels.app: apache`                Find Pods with the `app: apache`
                                           label

  `template.metadata.labels.app: apache`   Give new Pods the matching label

  `image: httpd:latest`                    Use the Apache container image

  `containerPort: 80`                      Apache listens on port 80 inside
                                           the container

  `requests.cpu: 100m`                     Set the CPU request

  `limits.cpu: 200m`                       Set the CPU limit
  ----------------------------------------------------------------------------

**Important:** The selector labels and Pod template labels must match.
Also, the field is `replicas`, not `replica`.

## 10. Create the Apache Service

Create `service.yaml`:

``` yaml
apiVersion: v1
kind: Service
metadata:
  name: apache-service
  namespace: apache-ns
spec:
  selector:
    app: apache
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: ClusterIP
```

Apply it:

``` bash
kubectl apply -f service.yaml
```

Check it:

``` bash
kubectl get service -n apache-ns
```

### Understand the ports

``` yaml
port: 80
targetPort: 80
```

-   `port` is the port exposed by the Service.
-   `targetPort` is the port on the application container.

The Service selects Pods with this label:

``` yaml
app: apache
```

This must match the label on the Apache Pods.

## 11. Test Apache inside the cluster

You can test the Service from another Pod.

For example, run a temporary BusyBox Pod:

``` bash
kubectl run test-client \
  -n apache-ns \
  --image=busybox:1.36 \
  --restart=Never \
  -- sleep 3600
```

Open a shell inside it:

``` bash
kubectl exec -it test-client -n apache-ns -- sh
```

From inside the shell, run:

``` sh
wget -q -O- http://apache-service
```

You should see the Apache welcome page HTML.

Exit the shell:

``` sh
exit
```

Delete the temporary test Pod:

``` bash
kubectl delete pod test-client -n apache-ns
```

This test proves that the Service name resolves inside the cluster and
can reach the Apache application.

## 12. Create the Apache HPA

Create `hpa.yaml`:

``` yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: apache-hpa
  namespace: apache-ns
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: apache-deployment
  minReplicas: 1
  maxReplicas: 5
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 5
```

Apply it:

``` bash
kubectl apply -f hpa.yaml
```

Check it:

``` bash
kubectl get hpa -n apache-ns
```

Watch it continuously:

``` bash
kubectl get hpa -n apache-ns -w
```

Press `Ctrl + C` to stop watching.

### Understand the HPA YAML

  -----------------------------------------------------------------------
  Field                               Meaning
  ----------------------------------- -----------------------------------
  `scaleTargetRef`                    The resource HPA should scale

  `kind: Deployment`                  HPA will scale a Deployment

  `name: apache-deployment`           The Deployment to scale

  `minReplicas: 1`                    Do not scale below one Pod

  `maxReplicas: 5`                    Do not scale above five Pods

  `name: cpu`                         Use CPU metrics

  `averageUtilization: 5`             Aim for average CPU utilization of
                                      5% of the CPU requests
  -----------------------------------------------------------------------

The 5% target is deliberately very low for this learning experiment. It
makes it easier to trigger scaling. It is **not** a production
recommendation.

## 13. Generate traffic for Apache

Now we need to make Apache busy.

We will use BusyBox as a load generator. It repeatedly sends HTTP
requests to the Apache Service.

First, create the load-generator Pod:

``` bash
kubectl run load-generator \
  -n apache-ns \
  --image=busybox:1.36 \
  --restart=Never \
  -- /bin/sh -c 'for i in $(seq 1 5); do while true; do wget -q -O- http://apache-service >/dev/null; done & done; wait'
```

### Understand this command

  Part                           Simple meaning
  ------------------------------ ---------------------------------------------------------
  `kubectl run load-generator`   Create a Pod named `load-generator`
  `-n apache-ns`                 Put it in the Apache namespace
  `--image=busybox:1.36`         Use the small BusyBox image
  `--restart=Never`              Create a Pod, not a Deployment
  `while true`                   Keep repeating forever
  `wget`                         Send an HTTP request
  `http://apache-service`        Send the request to the Apache Service
  `-q`                           Do not show normal download messages
  `-O-`                          Write the response to standard output
  `>/dev/null`                   Throw away the response body
  `&`                            Run this loop in the background
  `wait`                         Keep the main shell running while background loops work

The `for` loop starts five background request loops. This creates
repeated traffic for testing.

**Remember:** Five loops do not guarantee five requests per second. The
actual request rate depends on the application, network, and machine.

## 14. Watch Apache scale up

Open a second terminal and run:

``` bash
kubectl get hpa -n apache-ns -w
```

In another terminal, check the Pods:

``` bash
kubectl get pods -n apache-ns -w
```

You can also check CPU usage:

``` bash
kubectl top pods -n apache-ns
```

You may see CPU utilization rise above the 5% target. HPA can then
increase the number of Apache Pods.

For example, your experiment showed the Deployment scaling up until it
reached its maximum of five replicas.

The exact CPU percentages and timing can be different on another
computer. That is normal.

### What does `147%/5%` mean?

If HPA shows:

``` text
cpu: 147%/5%
```

It means:

-   Current average CPU utilization is 147% of the CPU requests.
-   The target is 5% of the CPU requests.

It does **not** mean 147% of the CPU limit.

## 15. Stop Apache traffic and watch it scale down

Delete the load-generator Pod:

``` bash
kubectl delete pod load-generator -n apache-ns
```

Now watch HPA again:

``` bash
kubectl get hpa -n apache-ns -w
```

And watch the Pods:

``` bash
kubectl get pods -n apache-ns -w
```

When CPU usage decreases, HPA can reduce the replica count.

It may not happen immediately. HPA checks metrics periodically and uses
scaling rules and stabilization behavior to avoid reacting too
aggressively to short changes.

**Important:** Deleting the load-generator does not directly delete
Apache Pods. It only stops the extra traffic. HPA later observes lower
CPU usage and decides whether to scale down.

## 16. Useful Apache commands

``` bash
# Show the Deployment
kubectl get deployment -n apache-ns

# Show Pods
kubectl get pods -n apache-ns

# Show the Service
kubectl get service -n apache-ns

# Show HPA
kubectl get hpa -n apache-ns

# Watch HPA
kubectl get hpa -n apache-ns -w

# Show CPU and memory usage
kubectl top pods -n apache-ns

# Explain HPA events and current status
kubectl describe hpa apache-hpa -n apache-ns

# Show Deployment details
kubectl describe deployment apache-deployment -n apache-ns

# Manually scale the Deployment
kubectl scale deployment apache-deployment -n apache-ns --replicas=3
```

**Note:** If HPA is managing the Deployment, it may change the replica
count again after a manual scale command. For this reason, manual
scaling is mainly useful for learning or temporary testing.

------------------------------------------------------------------------

# Project 2: Django Notes App

## 17. What are we building?

The second project uses a real application instead of a simple web
server.

The application is a Django Notes App. It runs in Kubernetes and
connects to a MySQL database.

The Kubernetes resources are:

-   Namespace: `notes-app`
-   Deployment: `notes-app-deployment`
-   Service: `notes-app-service`
-   HPA: `notes-app`
-   Load generator: `load-generator`

The application image used in this practice was:

``` text
mudasirirshad/notes-app-k8s:latest
```

The Django application listens on port `8080`.

The MySQL database is outside the Django Deployment and is reached at:

``` text
192.168.49.1:3306
```

The database address and credentials in your own YAML must match your
environment. Do not publish real passwords or secrets in a public GitHub
repository.

### How this project works

``` text
BusyBox load-generator
          |
          | repeated HTTP requests
          v
  notes-app-service
          |
          +------------------+
          |                  |
          v                  v
   Django Pod 1        Django Pod 2
          |                  |
          +--------+---------+
                   |
                   v
             MySQL database
```

The Service sends requests to the Django Pods. Each Django Pod connects
to the MySQL database.

HPA scales the Django application Pods. It does not scale the MySQL
database.

## 18. Notes App Deployment

Your Deployment uses the following important settings:

``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: notes-app-deployment
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
          command:
            - python
            - manage.py
            - runserver
            - 0.0.0.0:8080
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 200m
              memory: 256Mi
```

Apply your Deployment file:

``` bash
kubectl apply -f deployment.yaml
```

Check it:

``` bash
kubectl get deployment -n notes-app
kubectl get pods -n notes-app
```

### What is `0.0.0.0:8080`?

It tells Django to listen on all network interfaces inside the container
on port 8080.

If Django listens only on `127.0.0.1`, other Pods may not be able to
reach it through the Pod network.

**Production note:** Django's built-in `runserver` is useful for local
learning, but it is not recommended for production. A production
deployment should use a production application server such as Gunicorn,
with suitable security and configuration.

## 19. Notes App Service

The Service exposes port 8080 and sends traffic to the container's port
8080.

The important configuration is:

``` yaml
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

Apply it:

``` bash
kubectl apply -f service.yaml
```

Check it:

``` bash
kubectl get service -n notes-app
```

The Service selects Pods with:

``` yaml
app: notes-app
```

That label must match the Django Pods.

## 20. Notes App HPA

The HPA used in this practice was:

``` yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: notes-app
  namespace: notes-app
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: notes-app-deployment
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 30
  minReplicas: 1
  maxReplicas: 5
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 20
```

Apply it:

``` bash
kubectl apply -f hpa.yaml
```

Check it:

``` bash
kubectl get hpa -n notes-app
```

Watch it:

``` bash
kubectl get hpa -n notes-app -w
```

### What is different from the Apache HPA?

  Setting                       Apache                Notes App
  -------------- --------------------- ------------------------
  Minimum Pods                       1                        1
  Maximum Pods                       5                        5
  CPU target                        5%                      20%
  Scale target     `apache-deployment`   `notes-app-deployment`
  Namespace                `apache-ns`              `notes-app`

The Notes App target is 20%. Its CPU request is 100m, so 20% corresponds
to an average of 20m CPU per Pod.

### What does the stabilization window do?

This setting:

``` yaml
behavior:
  scaleDown:
    stabilizationWindowSeconds: 30
```

tells HPA to consider recent metrics over a 30-second period when
deciding to scale down.

Think of it as a short waiting period. If traffic drops for just a
moment and then rises again, HPA is less likely to remove Pods too
quickly.

This is useful because real application traffic can go up and down.

## 21. Generate traffic for the Notes App

First, remove an old load-generator Pod if it is still running:

``` bash
kubectl delete pod load-generator -n notes-app --ignore-not-found
```

Now start the load generator:

``` bash
kubectl run load-generator \
  -n notes-app \
  --image=busybox:1.36 \
  --restart=Never \
  -- /bin/sh -c 'for i in $(seq 1 30); do while true; do wget -q -O- http://notes-app-service >/dev/null; done & done; wait'
```

This command starts 30 background loops. Each loop repeatedly sends
requests to the Notes App Service.

The request destination is:

``` text
http://notes-app-service
```

Kubernetes DNS resolves the Service name from within the same namespace.

Check that the load-generator Pod is running:

``` bash
kubectl get pods -n notes-app
```

## 22. Watch the Notes App scale up

In another terminal, run:

``` bash
kubectl get hpa -n notes-app -w
```

Watch the application Pods:

``` bash
kubectl get pods -n notes-app -w
```

Check CPU usage:

``` bash
kubectl top pods -n notes-app
```

During the practice, the Notes App HPA showed values similar to:

``` text
cpu: 11%/20%   replicas: 2
cpu: 16%/20%   replicas: 2
cpu: 23%/20%   replicas: 2
cpu: 23%/20%   replicas: 3
cpu: 26%/20%   replicas: 3
cpu: 26%/20%   replicas: 4
```

This showed the application scaling from two Pods to three and then four
as CPU utilization increased.

Your results may be different. CPU usage depends on your laptop, VM
resources, application behavior, and request rate.

## 23. Stop traffic and watch the Notes App scale down

Delete the load-generator:

``` bash
kubectl delete pod load-generator -n notes-app
```

Watch HPA:

``` bash
kubectl get hpa -n notes-app -w
```

Watch Pods:

``` bash
kubectl get pods -n notes-app -w
```

During the practice, the HPA later showed values similar to:

``` text
cpu: 14%/20%   replicas: 4
cpu: 15%/20%   replicas: 3
cpu: 10%/20%   replicas: 3
cpu: 10%/20%   replicas: 2
```

This demonstrated scale-down after the traffic stopped and CPU usage
decreased.

Again, the exact values and timing are not guaranteed to be the same on
every run.

## 24. Why does HPA scale the Django app but not MySQL?

This is an important architecture concept.

Your HPA points to:

``` yaml
scaleTargetRef:
  kind: Deployment
  name: notes-app-deployment
```

Therefore, HPA changes the replica count of the Django Deployment.

It does not manage the MySQL database because MySQL is not the target of
this HPA.

Think of it this way:

-   Django is the team of workers serving customers.
-   MySQL is the shared storage room where the workers get and save
    information.
-   Hiring more Django workers does not automatically create more
    storage rooms.

Scaling a database needs its own design. Databases often need careful
handling of storage, replication, consistency, and connections. Do not
simply increase database replicas without understanding the database
architecture.

------------------------------------------------------------------------

## 25. Understanding HPA output

Run:

``` bash
kubectl get hpa -n notes-app
```

You may see output similar to:

``` text
NAME        REFERENCE                         TARGETS    MINPODS   MAXPODS   REPLICAS
notes-app   Deployment/notes-app-deployment   23%/20%    1         5         3
```

Let's understand each column.

  Column        Meaning
  ------------- -----------------------------------------
  `NAME`        Name of the HPA
  `REFERENCE`   Resource HPA is controlling
  `TARGETS`     Current metric compared with the target
  `MINPODS`     Minimum allowed Pod count
  `MAXPODS`     Maximum allowed Pod count
  `REPLICAS`    Current number of replicas

### How to read `23%/20%`

-   `23%` is the current average CPU utilization as a percentage of CPU
    requests.
-   `20%` is the target utilization.

Because 23% is higher than the 20% target, HPA may request more
replicas.

If the current utilization is below the target, HPA may reduce replicas,
subject to its rules.

### Why does HPA not always scale immediately?

HPA does not necessarily change replicas after every single request.

It:

1.  Reads metrics.
2.  Compares the current value with the target.
3.  Calculates a desired replica count.
4.  Applies minimum and maximum replica limits.
5.  Follows scaling behavior and stabilization rules.

A simplified scaling formula is:

``` text
desired replicas =
ceil(current replicas × current metric / target metric)
```

For example, suppose:

``` text
Current replicas = 2
Current CPU       = 40%
Target CPU        = 20%
```

Then:

``` text
desired replicas = ceil(2 × 40 / 20)
                 = ceil(4)
                 = 4
```

This is a simplified example. Kubernetes also applies tolerance,
missing-metric handling, scaling policies, stabilization windows, and
min/max limits. Therefore, actual scaling can be different.

------------------------------------------------------------------------

## 26. Troubleshooting

### Problem: `kubectl top pods` says metrics are unavailable

Check Metrics Server:

``` bash
kubectl get pods -n kube-system
```

Enable it in Minikube if needed:

``` bash
minikube addons enable metrics-server
```

Wait a little, then retry:

``` bash
kubectl top pods -n notes-app
```

### Problem: HPA shows `<unknown>` for CPU

Check these things:

1.  Is Metrics Server running?
2.  Does the application container have a CPU request?
3.  Is the Pod running and ready?
4.  Does the HPA target the correct Deployment?
5.  Is the HPA in the same namespace as the target Deployment?

Useful command:

``` bash
kubectl describe hpa notes-app -n notes-app
```

Read the `Conditions` and `Events` sections. They often explain why
metrics or scaling are not working.

### Problem: HPA exists but the number of Pods does not change

Check:

``` bash
kubectl describe hpa notes-app -n notes-app
kubectl top pods -n notes-app
kubectl get pods -n notes-app
```

Possible reasons:

-   CPU usage is not above the target.
-   The load generator is not running.
-   The load generator is sending requests to the wrong Service.
-   The application is not doing enough CPU work per request.
-   HPA has already reached `maxReplicas`.
-   Metrics have not updated yet.

### Problem: The load-generator Pod is not running

Check its details:

``` bash
kubectl describe pod load-generator -n notes-app
```

Check its logs:

``` bash
kubectl logs load-generator -n notes-app
```

Check that the Service exists:

``` bash
kubectl get service -n notes-app
```

### Problem: New Pods are stuck in Pending

Check:

``` bash
kubectl describe pods -n notes-app
kubectl describe nodes
```

Your Minikube VM may not have enough CPU or memory to run additional
Pods.

Remember: HPA can request more Pods, but it cannot create more node
capacity by itself.

------------------------------------------------------------------------

## 27. Practice exercises

Try these exercises yourself. Do not just copy commands---first predict
what Kubernetes will do.

### Exercise 1: Understand the HPA limits

In the Apache HPA, change:

``` yaml
minReplicas: 1
maxReplicas: 5
```

to:

``` yaml
minReplicas: 2
maxReplicas: 4
```

Apply the file:

``` bash
kubectl apply -f hpa.yaml
```

Questions:

-   What is the minimum number of Apache Pods now?
-   Can HPA create five Pods?
-   What happens when CPU usage becomes very low?

### Exercise 2: Change the CPU target

Change the Notes App CPU target from:

``` yaml
averageUtilization: 20
```

to:

``` yaml
averageUtilization: 50
```

Apply the file and generate traffic again.

Observe whether it is easier or harder to trigger scaling.

Hint: A higher target means Pods must become more CPU-utilized before
HPA decides that more replicas are needed.

### Exercise 3: Watch CPU usage

Run:

``` bash
kubectl top pods -n notes-app
```

Questions:

-   Which Pod is using the most CPU?
-   Does CPU usage change when the load generator starts?
-   What happens after the load generator is deleted?

### Exercise 4: Inspect HPA events

Run:

``` bash
kubectl describe hpa notes-app -n notes-app
```

Find the `Events` section.

Questions:

-   Did HPA successfully read CPU metrics?
-   Did it scale the Deployment?
-   Are there any warning messages?

### Exercise 5: Test the maximum replica count

Keep the HPA maximum at five:

``` yaml
maxReplicas: 5
```

Generate traffic and watch:

``` bash
kubectl get hpa -n notes-app -w
```

Question:

-   What happens when the HPA reaches five replicas but CPU is still
    above the target?

Answer: HPA cannot scale beyond the configured maximum. It will remain
at five until the metrics or configuration change.

### Exercise 6: Understand CPU requests

Temporarily inspect your Deployment:

``` bash
kubectl describe deployment notes-app-deployment -n notes-app
```

Find the CPU request and CPU limit.

Questions:

-   What is the CPU request?
-   What is the CPU limit?
-   Which one is used as the reference for `averageUtilization`?

Answer: The CPU request is the reference for CPU utilization percentage.
The CPU limit is the maximum CPU the container is allowed to use.

------------------------------------------------------------------------

## 28. Clean up the practice resources

Only run these commands if you want to delete the project resources.

### Delete Apache resources

``` bash
kubectl delete namespace apache-ns
```

### Delete Notes App resources

``` bash
kubectl delete namespace notes-app
```

Deleting a namespace deletes the resources inside that namespace,
including Deployments, Pods, Services, and HPAs.

**Warning:** Do not delete the Notes App namespace if you still need the
application or its resources.

------------------------------------------------------------------------

## 29. Quick command reference

  ---------------------------------------------------------------------------------------------------------------
  Task                                Command
  ----------------------------------- ---------------------------------------------------------------------------
  List namespaces                     `kubectl get namespaces`

  List Pods                           `kubectl get pods -n notes-app`

  List Deployments                    `kubectl get deployments -n notes-app`

  List Services                       `kubectl get services -n notes-app`

  List HPA                            `kubectl get hpa -n notes-app`

  Watch HPA                           `kubectl get hpa -n notes-app -w`

  Watch Pods                          `kubectl get pods -n notes-app -w`

  View Pod CPU/memory                 `kubectl top pods -n notes-app`

  View node CPU/memory                `kubectl top nodes`

  Inspect HPA                         `kubectl describe hpa notes-app -n notes-app`

  View Pod details                    `kubectl describe pod <pod-name> -n notes-app`

  View Pod logs                       `kubectl logs <pod-name> -n notes-app`

  Delete load generator               `kubectl delete pod load-generator -n notes-app`

  Manually scale Deployment           `kubectl scale deployment notes-app-deployment -n notes-app --replicas=3`
  ---------------------------------------------------------------------------------------------------------------

In your terminal, you use `k` as an alias for `kubectl`. For example:

``` bash
k get pods -n notes-app
```

is the same as:

``` bash
kubectl get pods -n notes-app
```

The README uses the full `kubectl` command so that it is easy for other
people to follow.

------------------------------------------------------------------------

## 30. What you should remember

The complete lesson can be summarized in a few points:

1.  A **Pod** runs your application.
2.  A **Deployment** manages the number of application Pods.
3.  A **Service** gives clients one stable address to reach the Pods.
4.  **Metrics Server** collects CPU and memory usage metrics.
5.  **HPA** reads metrics and changes the Deployment's replica count.
6.  **CPU requests** are important for CPU utilization-based HPA.
7.  A **load generator** creates traffic so you can test scaling.
8.  HPA scales application Pods; it does not automatically scale your
    database or add cluster nodes.
9.  HPA may take time to scale up or down because it follows metrics
    intervals and scaling behavior.
10. The exact results depend on your machine, application, resource
    settings, and traffic.

## Final result

You practiced HPA with two different workloads:

-   **Apache:** You used a simple web server to understand the basic
    autoscaling flow.
-   **Django Notes App:** You repeated the same idea with a real
    application connected to MySQL.

The most important thing is not memorizing the YAML. It is understanding
this flow:

``` text
Traffic increases
       |
       v
Application CPU usage increases
       |
       v
Metrics Server reports the usage
       |
       v
HPA compares usage with the target
       |
       v
Deployment gets a higher replica count
       |
       v
More application Pods start
```

When traffic decreases, the process works in the opposite direction and
HPA can reduce the number of Pods.

That is the basic idea of **Horizontal Pod Autoscaling in Kubernetes**.
