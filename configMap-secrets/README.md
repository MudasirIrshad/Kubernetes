# Kubernetes ConfigMaps and Secrets --- A Beginner-Friendly Guide

> **Goal:** Understand ConfigMaps, Secrets, StatefulSets, PVCs, and how
> they work together with MySQL.
>
> This guide uses simple language, everyday examples, practical YAML,
> terminal commands, and small experiments. You do not need to
> understand everything at once. Follow the sections in order and try
> the commands yourself.

------------------------------------------------------------------------

## 1. The Big Picture: What Are We Building?

Imagine you are running a small restaurant.

-   The **restaurant building** is like your Kubernetes Pod.
-   The **recipe card** tells the cook what food to prepare. This is
    like a ConfigMap.
-   The **safe containing the cash-box key** is like a Secret.
-   The **cook** is like the MySQL container.
-   The **storage room** where ingredients are kept is like persistent
    storage.
-   The **restaurant manager** who makes sure the right number of cooks
    are working is like a StatefulSet controller.

In our Kubernetes practice, we are running MySQL in a namespace called
`mysql-namespace`.

Our setup includes:

-   A ConfigMap containing the database name: `devops`.
-   A Secret containing the MySQL root password.
-   A headless Service named `mysql-service`.
-   A StatefulSet named `mysql-statefulset`.
-   Three MySQL Pods.
-   One PersistentVolumeClaim (PVC) for each Pod.

**Important:** These three MySQL Pods are three separate MySQL servers.
They do not automatically share or replicate their databases. A
StatefulSet gives them stable identities and individual storage; it does
not turn them into a database cluster.

------------------------------------------------------------------------

## 2. What Is a Kubernetes Namespace?

A namespace is like a separate room or department inside a large
building.

A company might have: - A Finance department. - A Sales department. - An
Engineering department.

Each department keeps its own work organized. Kubernetes namespaces help
organize resources in a cluster.

We used:

``` yaml
apiVersion: v1
kind: Namespace
metadata:
  name: mysql-namespace
```

Save this as `namespace.yaml`, then apply it:

``` bash
kubectl apply -f namespace.yaml
```

If you use the alias `k` for `kubectl`, you can run:

``` bash
k apply -f namespace.yaml
```

Check namespaces:

``` bash
k get namespaces
```

**Remember:** A namespace is an organizational boundary. It is not a
separate Kubernetes cluster, and it does not automatically provide
complete security isolation.

------------------------------------------------------------------------

## 3. What Is a ConfigMap?

### Imagine a school lunchbox

A parent packs a lunchbox and puts a note inside:

``` text
Today's lunch: Sandwich
Drink: Milk
```

The child can read the note and know what to eat. The note is separate
from the lunchbox itself.

A ConfigMap works in a similar way. It stores ordinary, non-sensitive
configuration separately from the application container.

For our MySQL example, we want the database name to be `devops`.

### Our `configMap.yaml`

``` yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql-config-map
  namespace: mysql-namespace
data:
  MYSQL_DATABASE: devops
```

Apply it:

``` bash
k apply -f configMap.yaml
```

Check it:

``` bash
k get configmaps -n mysql-namespace
```

See its contents:

``` bash
k get configmap mysql-config-map -n mysql-namespace -o yaml
```

You should find:

``` yaml
data:
  MYSQL_DATABASE: devops
```

### Why use a ConfigMap?

Suppose tomorrow you want the database name to be `school_db` instead of
`devops`.

You can change the ConfigMap instead of putting that setting directly
into your application image.

### What belongs in a ConfigMap?

Examples:

-   Database name.
-   Application mode, such as `development` or `production`.
-   A display setting.
-   A non-secret service URL.
-   A feature setting.

**Never put passwords, API keys, or private tokens in a ConfigMap.** Use
a Secret for sensitive values.

------------------------------------------------------------------------

## 4. What Is a Secret?

### Imagine a locked diary

You might write ordinary notes on a piece of paper, but you would keep
your house key or bank PIN somewhere more protected.

A Kubernetes Secret is designed to hold sensitive configuration, such
as:

-   Passwords.
-   API tokens.
-   Certificates.
-   Private keys.

For our example, the MySQL root password is `root`. This is only a
simple practice password. Do not use it in a real or production
environment.

### Create a Secret using a command

``` bash
k create secret generic mysql-secret \
  --from-literal=MYSQL_ROOT_PASSWORD=root \
  -n mysql-namespace
```

Check that it exists:

``` bash
k get secrets -n mysql-namespace
```

See the Secret's keys (without printing their values):

``` bash
k describe secret mysql-secret -n mysql-namespace
```

### Why does the value look encoded?

Kubernetes Secret data is commonly displayed using Base64 encoding.

For example, to decode our practice value:

``` bash
k get secret mysql-secret -n mysql-namespace \
  -o jsonpath='{.data.MYSQL_ROOT_PASSWORD}' | base64 -d
echo
```

Output:

``` text
root
```

**Security note:** Base64 is encoding, not encryption. A Secret is not
automatically safe just because its value is Base64-encoded. Real
clusters should use appropriate access controls (RBAC), encryption at
rest, and careful handling of credentials. Avoid printing real passwords
in terminal recordings, screenshots, or shared logs.

### Secret YAML alternative

You can also define a Secret in YAML, but the `data` field expects
Base64-encoded values. For learning, creating it with
`kubectl create secret generic` avoids manually encoding the value.

Do not commit real Secret values to a public Git repository.

------------------------------------------------------------------------

## 5. ConfigMap vs Secret

Think of a school bag:

-   The timetable is ordinary information. It is like a ConfigMap.
-   The key to a locker is sensitive. It is like a Secret.

  -----------------------------------------------------------------------------
  Feature                 ConfigMap                 Secret
  ----------------------- ------------------------- ---------------------------
  Main purpose            Ordinary configuration    Sensitive configuration

  Example                 `MYSQL_DATABASE=devops`   `MYSQL_ROOT_PASSWORD=...`

  Common use              App settings              Passwords and tokens

  Stored as               Text data or binary data  Base64-encoded data by
                                                    default

  Automatically           No                        Not by default; configure
  encrypted?                                        encryption at rest
  -----------------------------------------------------------------------------

A Secret is intended for sensitive information, but you still need to
configure and protect your cluster properly.

------------------------------------------------------------------------

## 6. How Does a Pod Receive ConfigMap and Secret Values?

A Pod is like a small work area where your application runs.

Kubernetes can place configuration into a container as environment
variables.

In our StatefulSet, the MySQL container receives two variables:

``` yaml
env:
  - name: MYSQL_ROOT_PASSWORD
    valueFrom:
      secretKeyRef:
        name: mysql-secret
        key: MYSQL_ROOT_PASSWORD

  - name: MYSQL_DATABASE
    valueFrom:
      configMapKeyRef:
        name: mysql-config-map
        key: MYSQL_DATABASE
```

Read it slowly:

### First variable: password

``` yaml
- name: MYSQL_ROOT_PASSWORD
  valueFrom:
    secretKeyRef:
      name: mysql-secret
      key: MYSQL_ROOT_PASSWORD
```

-   `name: MYSQL_ROOT_PASSWORD` is the environment variable name inside
    the container.
-   `secretKeyRef` tells Kubernetes to read from a Secret.
-   `name: mysql-secret` is the Kubernetes Secret resource name.
-   `key: MYSQL_ROOT_PASSWORD` is the key inside that Secret.

### Second variable: database name

``` yaml
- name: MYSQL_DATABASE
  valueFrom:
    configMapKeyRef:
      name: mysql-config-map
      key: MYSQL_DATABASE
```

This works similarly, but it reads from a ConfigMap.

### Verify the variables inside a Pod

``` bash
k exec mysql-statefulset-0 -n mysql-namespace -- printenv MYSQL_DATABASE
```

Expected:

``` text
devops
```

Check the password variable without displaying its value in shared
output:

``` bash
k exec mysql-statefulset-0 -n mysql-namespace -- sh -c \
  'test -n "$MYSQL_ROOT_PASSWORD" && echo "Password variable is set"'
```

**Important distinction:** The Kubernetes environment variable is
configuration supplied to the container. It does not mean Kubernetes
itself has created a database or changed a password inside MySQL.

------------------------------------------------------------------------

## 7. How the MySQL Docker Image Uses These Variables

This is a very important part of our practical exercise.

The official MySQL Docker image has an entrypoint script. When the
container starts, this script checks the MySQL data directory.

Our StatefulSet mounts storage here:

``` yaml
volumeMounts:
  - name: mysql-data
    mountPath: /var/lib/mysql
```

Think of `/var/lib/mysql` as MySQL's notebook. It stores the database
files and information needed to run the database.

### First initialization

When the data directory is empty, the official image performs its
first-time initialization. During this process, it can use variables
such as:

-   `MYSQL_ROOT_PASSWORD` to configure the initial root password.
-   `MYSQL_DATABASE` to create an initial database.

For our example, the intended result is a database named `devops`.

### Later restarts

After MySQL has initialized its data directory, it finds existing
database files on the next start.

It does not repeat the first-time setup just because you changed an
environment variable.

For example:

1.  MySQL initializes with `MYSQL_DATABASE=devops`.
2.  The database files are saved in the PVC.
3.  You change the ConfigMap to `MYSQL_DATABASE=school_db`.
4.  The Pod restarts.
5.  MySQL sees an existing data directory and does not automatically
    create `school_db`.

The same principle applies to the root password: changing the Secret
does not automatically change the password of an already initialized
MySQL account.

**Key lesson:** Environment variables can configure an application's
initial setup, but they are not a universal way to modify an
application's existing internal data.

------------------------------------------------------------------------

## 8. Why Didn't Our `devops` Database Appear Automatically?

In our practice session, we confirmed all of the following:

-   The ConfigMap contained `MYSQL_DATABASE: devops`.
-   The MySQL container received `MYSQL_DATABASE=devops`.
-   The StatefulSet mounted a PVC at `/var/lib/mysql`.
-   MySQL started and became ready.
-   `SHOW DATABASES;` initially did not show `devops`.

We then created it manually:

``` sql
CREATE DATABASE IF NOT EXISTS devops;
```

And verified it:

``` sql
SHOW DATABASES;
```

It appeared successfully.

### What do we know, and what do we not know?

We know that Kubernetes supplied the environment variable correctly. We
also know that MySQL was running.

However, the logs and outputs we collected did not establish exactly why
the official MySQL initialization process did not create the database
automatically. We should not pretend that the cause was proven.

If this happens again in a fresh test, investigate the image's
entrypoint and initialization logs, confirm that the data directory was
genuinely empty at first startup, and check whether initialization
scripts or custom commands changed the normal startup process.

Do not repeatedly delete PVCs without a reason. Deleting them can erase
database data.

------------------------------------------------------------------------

## 9. How to Create a Database Manually

If MySQL is already running and initialized, connect to it and execute
SQL.

Enter the Pod:

``` bash
k exec -it mysql-statefulset-0 -n mysql-namespace -- bash
```

Connect to MySQL:

``` bash
mysql -u root -p
```

Enter the root password when prompted.

Create the database:

``` sql
CREATE DATABASE IF NOT EXISTS devops;
```

List databases:

``` sql
SHOW DATABASES;
```

Select the database:

``` sql
USE devops;
```

Create a sample table:

``` sql
CREATE TABLE students (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100),
    age INT
);
```

Insert a row:

``` sql
INSERT INTO students (name, age)
VALUES ('Ali', 10);
```

Read the row:

``` sql
SELECT * FROM students;
```

You have now created a database and stored a record in it.

Exit MySQL:

``` sql
exit;
```

Then exit the container:

``` bash
exit
```

### A note about password commands

This command can pass the environment variable directly:

``` bash
mysql -u root -p"$MYSQL_ROOT_PASSWORD"
```

It may show a warning because passwords supplied on the command line can
be exposed through process information or other diagnostics. For
interactive work, prefer:

``` bash
mysql -u root -p
```

Then type the password at the prompt. Never paste a real password into a
shared screenshot or public log.

------------------------------------------------------------------------

## 10. What Is a StatefulSet?

Imagine a school has three classrooms:

-   Classroom 0.
-   Classroom 1.
-   Classroom 2.

Each classroom has its own number and its own cupboard. If a teacher
leaves and is replaced, the classroom number and cupboard remain
associated with that classroom.

A StatefulSet provides similar stable identities to Pods.

Our StatefulSet is named:

``` text
mysql-statefulset
```

With three replicas, Kubernetes creates Pods named:

``` text
mysql-statefulset-0
mysql-statefulset-1
mysql-statefulset-2
```

Unlike randomly generated Deployment Pod names, StatefulSet Pod names
have stable ordinals.

### Why is this useful?

Databases and other stateful applications often need predictable
identities and persistent storage.

A StatefulSet helps provide:

-   Stable Pod names.
-   Stable network identity when used with the appropriate Service.
-   Individual persistent storage for each replica.
-   Ordered creation and termination behavior.

### StatefulSet YAML used in our lesson

``` yaml
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

          ports:
            - containerPort: 3306

          env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: MYSQL_ROOT_PASSWORD

            - name: MYSQL_DATABASE
              valueFrom:
                configMapKeyRef:
                  name: mysql-config-map
                  key: MYSQL_DATABASE

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

Save it as `statefulset.yaml`.

Apply it:

``` bash
k apply -f statefulset.yaml
```

Check Pods:

``` bash
k get pods -n mysql-namespace
```

Check the StatefulSet:

``` bash
k get statefulsets -n mysql-namespace
```

------------------------------------------------------------------------

## 11. What Is a Service?

Imagine a school has a reception desk. Visitors do not need to know
which classroom has a teacher available; they can contact the school
through the reception desk.

A Kubernetes Service gives a stable network endpoint for accessing
selected Pods.

Our StatefulSet uses a headless Service named `mysql-service`.

A headless Service is configured with:

``` yaml
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
    - port: 3306
      targetPort: 3306
```

Save this as `service.yaml`.

Apply it:

``` bash
k apply -f service.yaml
```

### Why is it called headless?

Because `clusterIP: None` means Kubernetes does not assign the Service a
normal virtual ClusterIP.

For StatefulSets, a headless Service can provide DNS records for
individual Pods.

A Pod's DNS name follows this pattern:

``` text
<pod-name>.<service-name>.<namespace>.svc.cluster.local
```

For our first Pod:

``` text
mysql-statefulset-0.mysql-service.mysql-namespace.svc.cluster.local
```

This helps applications discover specific StatefulSet members.

------------------------------------------------------------------------

## 12. What Is a PersistentVolumeClaim (PVC)?

Imagine you are a student who writes homework in a notebook.

If the student leaves the classroom and a replacement student arrives,
the notebook should not disappear.

A PVC is like a request for a storage space where data can be kept
beyond the life of an individual Pod.

Our StatefulSet uses:

``` yaml
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

This tells Kubernetes to create a separate PVC for each StatefulSet Pod.

For example:

``` text
mysql-data-mysql-statefulset-0
mysql-data-mysql-statefulset-1
mysql-data-mysql-statefulset-2
```

Check them:

``` bash
k get pvc -n mysql-namespace
```

### What does `1Gi` mean?

It requests approximately one gibibyte of storage for each claim.

### What does `ReadWriteOnce` mean?

It means the volume can be mounted read-write by a single node at a
time. It does not necessarily mean only one Pod can ever use it; the
exact behavior depends on the storage system.

### What happens if a Pod is deleted?

If `mysql-statefulset-0` is deleted, the StatefulSet controller normally
creates a replacement Pod with the same name.

Its PVC is retained, so the replacement can use the same persistent
data.

The Pod is recreated; the old Pod process itself is not brought back.

------------------------------------------------------------------------

## 13. Why Do We Have Three PVCs?

Our StatefulSet has:

``` yaml
replicas: 3
```

And a `volumeClaimTemplates` entry named `mysql-data`.

Kubernetes therefore creates one claim for each replica.

  Pod                     Its PVC
  ----------------------- ----------------------------------
  `mysql-statefulset-0`   `mysql-data-mysql-statefulset-0`
  `mysql-statefulset-1`   `mysql-data-mysql-statefulset-1`
  `mysql-statefulset-2`   `mysql-data-mysql-statefulset-2`

Each Pod has its own `/var/lib/mysql`.

**Very important:** These are independent MySQL databases. Creating a
database in Pod 0 does not automatically create it in Pod 1 or Pod 2.
Likewise, inserting a row into Pod 0 does not replicate it to the other
Pods.

A real highly available MySQL cluster requires database replication or a
database clustering solution. A StatefulSet alone does not provide that.

------------------------------------------------------------------------

## 14. Useful Commands We Practised

Assuming `k` is your alias for `kubectl`:

### Inspect resources

``` bash
k get namespaces
k get pods -n mysql-namespace
k get services -n mysql-namespace
k get statefulsets -n mysql-namespace
k get configmaps -n mysql-namespace
k get secrets -n mysql-namespace
k get pvc -n mysql-namespace
```

### Inspect details

``` bash
k describe pod mysql-statefulset-0 -n mysql-namespace
k describe statefulset mysql-statefulset -n mysql-namespace
k describe pvc mysql-data-mysql-statefulset-0 -n mysql-namespace
```

### View logs

``` bash
k logs mysql-statefulset-0 -n mysql-namespace
```

If a container restarted, view its previous logs:

``` bash
k logs mysql-statefulset-0 -n mysql-namespace --previous
```

### Enter a Pod

``` bash
k exec -it mysql-statefulset-0 -n mysql-namespace -- bash
```

### Run a command without entering the Pod

``` bash
k exec mysql-statefulset-0 -n mysql-namespace -- printenv MYSQL_DATABASE
```

### Apply YAML files

``` bash
k apply -f namespace.yaml
k apply -f configMap.yaml
k apply -f secrets.yaml
k apply -f service.yaml
k apply -f statefulset.yaml
```

Apply all YAML files in the current directory:

``` bash
k apply -f .
```

Only do this when every YAML file in that directory is intended to be
applied.

------------------------------------------------------------------------

## 15. Deleting Resources Safely

You can delete a namespace and its namespaced resources:

``` bash
k delete namespace mysql-namespace
```

But remember that persistent storage needs special care.

Deleting a StatefulSet does not necessarily delete its PVCs. StatefulSet
PVC retention settings and the storage class's reclaim policy affect
what happens to storage.

To list PVCs:

``` bash
k get pvc -n mysql-namespace
```

To delete all PVCs in that namespace:

``` bash
k delete pvc --all -n mysql-namespace
```

**Warning:** Deleting PVCs can permanently destroy database data,
depending on the storage and reclaim policy. Never run this in a real
environment unless you have confirmed that the data can be discarded or
safely backed up.

For learning, use test data only.

------------------------------------------------------------------------

## 16. Common Problems We Encountered

### Problem A: `ImagePullBackOff`

**Meaning:** Kubernetes could not obtain the container image.

If you are using Minikube and the image is already available in your
computer's Docker environment, remember that Minikube may use a separate
image store.

Useful checks:

``` bash
k describe pod <pod-name> -n mysql-namespace
minikube image ls | grep mysql
```

You may need to load the image into Minikube:

``` bash
minikube image load mysql:8.0
```

For a locally loaded image, an appropriate pull policy such as
`IfNotPresent` can be useful.

### Problem B: `ERROR 2002`

Example:

``` text
Can't connect to local MySQL server through socket
```

**Meaning:** The MySQL server may not be running yet, may still be
starting, or may not be reachable through that socket.

Check:

``` bash
k get pods -n mysql-namespace
k logs mysql-statefulset-0 -n mysql-namespace
```

Wait until the Pod is ready before connecting.

### Problem C: `ERROR 1045`

Example:

``` text
Access denied for user 'root'@'localhost'
```

**Meaning:** MySQL was reached, but it rejected the supplied
credentials.

Check that the Secret and environment variable contain the expected
value:

``` bash
k get secret mysql-secret -n mysql-namespace
k exec mysql-statefulset-0 -n mysql-namespace -- \
  printenv MYSQL_ROOT_PASSWORD
```

Remember: changing the Secret does not automatically change the password
of a MySQL account that has already been initialized.

### Problem D: Database is missing

Check the ConfigMap:

``` bash
k get configmap mysql-config-map -n mysql-namespace -o yaml
```

Check the environment variable:

``` bash
k exec mysql-statefulset-0 -n mysql-namespace -- \
  printenv MYSQL_DATABASE
```

Check databases in MySQL:

``` sql
SHOW DATABASES;
```

If MySQL has already initialized, create a missing database with SQL:

``` sql
CREATE DATABASE IF NOT EXISTS devops;
```

If the official image should have created it during first initialization
but did not, investigate the entrypoint and startup process rather than
assuming the ConfigMap is broken.

------------------------------------------------------------------------

## 17. Hands-On Practice Exercises

Try these exercises yourself. Do not just copy the commands---first
predict what you think will happen.

### Exercise 1: Change the database name in the ConfigMap

Change:

``` yaml
MYSQL_DATABASE: devops
```

to:

``` yaml
MYSQL_DATABASE: school_db
```

Apply it:

``` bash
k apply -f configMap.yaml
```

Check the Pod environment:

``` bash
k exec mysql-statefulset-0 -n mysql-namespace -- \
  printenv MYSQL_DATABASE
```

**Question:** Does changing the environment variable automatically
create `school_db` inside an already initialized MySQL database?

Expected learning: No. The environment variable and the database's
internal state are separate things.

### Exercise 2: Create a table

Connect to MySQL and run:

``` sql
USE devops;

CREATE TABLE books (
    id INT AUTO_INCREMENT PRIMARY KEY,
    title VARCHAR(100)
);

INSERT INTO books (title)
VALUES ('Learning Kubernetes');

SELECT * FROM books;
```

### Exercise 3: Inspect the three PVCs

``` bash
k get pvc -n mysql-namespace
```

Match each PVC to its StatefulSet Pod.

### Exercise 4: Observe stable Pod identity

Delete only the first Pod:

``` bash
k delete pod mysql-statefulset-0 -n mysql-namespace
```

Watch the Pods:

``` bash
k get pods -n mysql-namespace -w
```

Observe that the StatefulSet recreates `mysql-statefulset-0` and that
its PVC remains.

### Exercise 5: Compare the databases

Connect separately to Pod 0 and Pod 1.

Run:

``` sql
SHOW DATABASES;
```

If you created `devops` only in Pod 0, you should not expect it to
appear automatically in Pod 1.

This demonstrates why three StatefulSet replicas are not automatically a
replicated MySQL cluster.

------------------------------------------------------------------------

## 18. Quick Revision: Remember These Ideas

1.  **Namespace:** Organizes Kubernetes resources into groups.
2.  **ConfigMap:** Stores ordinary configuration.
3.  **Secret:** Stores sensitive configuration; protect it with proper
    security controls.
4.  **Pod:** Runs one or more containers.
5.  **StatefulSet:** Manages Pods with stable identities and storage
    associations.
6.  **Service:** Provides a stable way to discover and reach Pods.
7.  **Headless Service:** Supports direct DNS discovery of individual
    Pods.
8.  **PVC:** Requests persistent storage.
9.  **Environment variable:** Makes a configuration value available
    inside a container.
10. **MySQL entrypoint:** Uses initialization variables during
    first-time setup of an empty data directory.
11. **Persistent data:** Can survive Pod replacement when the same PVC
    is reused.
12. **Three MySQL Pods:** Do not automatically mean three synchronized
    copies of one database.

------------------------------------------------------------------------

## 19. What to Learn Next

A natural next topic is **Kubernetes Volumes and Persistent Storage**.

Learn these concepts next:

-   Volumes.
-   PersistentVolumes (PV).
-   PersistentVolumeClaims (PVC).
-   StorageClasses.
-   Dynamic provisioning.
-   Reclaim policies.
-   What happens to data when Pods, PVCs, and StatefulSets are deleted.

These topics will explain in greater depth how the storage in our MySQL
StatefulSet works.

------------------------------------------------------------------------

## Final Thought

Think of Kubernetes as the manager of a big school:

-   Namespaces organize classrooms.
-   Pods are classrooms where work happens.
-   ConfigMaps are instruction sheets.
-   Secrets are locked information.
-   StatefulSets give certain classrooms stable identities.
-   Services help people find the right classroom.
-   PVCs provide notebooks that should remain available even when a
    classroom is replaced.

Once you understand what each part is responsible for, Kubernetes
becomes much easier to reason about.
