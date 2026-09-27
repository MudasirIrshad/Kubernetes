# Kubernetes Jobs & CronJobs

Jobs and CronJobs are Kubernetes resources used for running tasks that need to **complete**, instead of applications that need to run continuously.

---

# 1. What is a Job?

A **Job** is a Kubernetes resource that runs a task **one time** and makes sure that the task completes successfully.

## Simple Analogy

Imagine you have a restaurant and you tell a worker:

> "Clean the kitchen."

The worker:

1. Starts the work.
2. Cleans the kitchen.
3. Finishes the work.
4. Stops after the work is completed.

This is similar to a Kubernetes **Job**.

```text
Job
 ↓
Pod
 ↓
Container
 ↓
Task
 ↓
Completed
````

A Job is useful when we have work that should **finish successfully**.

## Common Use Cases

Jobs are commonly used for:

* Database migrations
* Backups
* Data processing
* Batch processing
* Generating reports
* One-time scripts
* Initialization tasks
* Cleanup tasks

---

# 2. Job vs Deployment

A **Deployment** is normally used for applications that should keep running.

For example, if we deploy NGINX:

```text
Deployment
    ↓
Pod
    ↓
NGINX
    ↓
Keeps running
```

If the NGINX Pod crashes, Kubernetes can create another Pod.

A **Job** is different.

A Job runs a task and stops when the task is complete:

```text
Job
 ↓
Pod
 ↓
Run task
 ↓
Complete
```

## Easy Way to Remember

> **Deployment = Keep the application running**

> **Job = Complete the task**

---

# 3. Job Structure

The basic relationship is:

```text
Job
 │
 └── Pod
      │
      └── Container
           │
           └── Task
```

The Job controls the Pod and makes sure the required work is completed successfully.

---

# 4. Creating a Job

Create a file called:

```text
job.yml
```

Example:

```yaml
apiVersion: batch/v1
kind: Job

metadata:
  name: nginx-job
  namespace: nginx

spec:
  completions: 1
  parallelism: 1

  template:
    metadata:
      name: nginx-job-pod
      labels:
        app: batch-task

    spec:
      containers:
        - name: batch-container
          image: busybox:latest
          command: ["sh", "-c", "echo hello dosto && sleep 10"]

      restartPolicy: Never
```

## What does this Job do?

The Job creates a Pod.

The Pod runs:

```bash
echo hello dosto
```

Then waits for 10 seconds:

```bash
sleep 10
```

After the command finishes successfully:

```text
Pod → Completed
Job → Complete
```

---

# 5. Apply the Job

Apply the YAML file:

```bash
k apply -f job.yml
```

Example output:

```text
job.batch/nginx-job created
```

---

# 6. Check the Job

To see Jobs in the `nginx` namespace:

```bash
k get jobs -n nginx
```

Example:

```text
NAME        STATUS     COMPLETIONS   DURATION   AGE
nginx-job   Complete   1/1           26s        1m
```

## Understanding `1/1`

```text
1/1
```

means:

```text
1 successful completion
out of
1 required completion
```

So the Job has successfully completed its required work.

---

# 7. Check the Pod Created by the Job

A Job creates a Pod.

Check the Pods:

```bash
k get pods -n nginx
```

Example:

```text
NAME             READY   STATUS      RESTARTS   AGE
nginx-job-46grt  0/1     Completed   0          1m
```

The Pod is `Completed` because the Job's task has finished.

---

# 8. Check Which Node Runs the Job Pod

Use:

```bash
k get pods -n nginx -o wide
```

Example:

```text
NAME             READY   STATUS      NODE
nginx-job-46grt  0/1     Completed   minikube-m03
```

This tells us which Kubernetes Node ran the Pod.

---

# 9. View Job Logs

The actual application/container logs belong to the **Pod**.

First find the Pod:

```bash
k get pods -n nginx
```

Then:

```bash
k logs nginx-job-46grt -n nginx
```

You should see:

```text
hello dosto
```

You can also get logs through the Job:

```bash
k logs job/nginx-job -n nginx
```

---

# 10. What is a CronJob?

A **CronJob** is a Kubernetes resource that automatically creates **Jobs repeatedly according to a schedule**.

## Simple Analogy

Imagine you have a worker responsible for cleaning a restaurant.

You tell the worker:

> "Clean the kitchen every day at 10 PM."

You don't need to manually tell the worker every day.

The schedule automatically tells the worker when to perform the task.

This is similar to a Kubernetes **CronJob**.

```text
CronJob
   ↓
Creates Job
   ↓
Creates Pod
   ↓
Runs Task
   ↓
Completed
```

When the next scheduled time arrives, the CronJob creates another Job.

---

# 11. Job vs CronJob

The main difference is:

```text
Job
 ↓
Runs a task once
```

while:

```text
CronJob
 ↓
Automatically creates Jobs
according to a schedule
```

## Easy Definition

> **Job = One-time work**

> **CronJob = Repeated scheduled work**

---

# 12. CronJob Hierarchy

A very important concept is that a CronJob does **not** repeatedly reuse the same Pod.

Every scheduled execution creates a **new Job**.

That Job then creates a **new Pod**.

```text
                 CronJob
                    │
             Every 1 minute
                    │
        ┌───────────┼───────────┐
        ↓           ↓           ↓
      Job #1      Job #2      Job #3
        ↓           ↓           ↓
      Pod #1      Pod #2      Pod #3
        ↓           ↓           ↓
      Task        Task        Task
        ↓           ↓           ↓
   Completed   Completed   Completed
```

For example, if a CronJob runs every minute:

```text
10:01 → Job #1 → Pod #1 → Task
10:02 → Job #2 → Pod #2 → Task
10:03 → Job #3 → Pod #3 → Task
10:04 → Job #4 → Pod #4 → Task
```

So the important relationship is:

```text
CronJob → Job → Pod → Container → Task
```

---

# 13. Cron Expressions

CronJob uses a **cron expression** to define when a Job should run.

The basic format is:

```text
* * * * *
│ │ │ │ │
│ │ │ │ └── Day of week
│ │ │ └──── Month
│ │ └────── Day of month
│ └──────── Hour
└────────── Minute
```

## Common Cron Schedules

### Every minute

```text
* * * * *
```

### Every 5 minutes

```text
*/5 * * * *
```

### Every hour

```text
0 * * * *
```

### Every day at midnight

```text
0 0 * * *
```

---

# 14. CronJob Backup Example

For practice, we created a CronJob that performs a backup every minute.

Create:

```text
cron-job.yml
```

Example:

```yaml
apiVersion: batch/v1
kind: CronJob

metadata:
  name: cronjob-backup
  namespace: nginx

spec:
  schedule: "* * * * *"

  jobTemplate:
    spec:
      template:
        metadata:
          name: minute-backup
          labels:
            app: minute-backup

        spec:
          containers:
            - name: backup-container
              image: busybox
              command: ["sh", "-c", "echo Backup Started && mkdir -p /backups && mkdir -p /demo-data && cp -r /demo-data /backups && echo Backup completed"]

              volumeMounts:
                - name: data-volume
                  mountPath: /demo-data

                - name: backup-volume
                  mountPath: /backups

          restartPolicy: OnFailure

          volumes:
            - name: data-volume
              hostPath:
                path: /demo-data
                type: DirectoryOrCreate

            - name: backup-volume
              hostPath:
                path: /backup
                type: DirectoryOrCreate
```

The important part is:

```yaml
schedule: "* * * * *"
```

This means:

> Run the Job every minute.

---

# 15. Understanding `jobTemplate`

A CronJob contains a **Job template**.

```yaml
spec:
  schedule: "* * * * *"

  jobTemplate:
    spec:
      template:
```

Think of the CronJob as a factory.

The factory knows:

> "Every minute, create a Job using these instructions."

The flow is:

```text
CronJob
   │
   │ Every minute
   ↓
Creates Job
   │
   ↓
Creates Pod
   │
   ↓
Runs Task
```

---

# 16. Apply the CronJob

Apply the YAML:

```bash
k apply -f cron-job.yml
```

Example output:

```text
cronjob.batch/cronjob-backup created
```

---

# 17. Check the CronJob

Because the CronJob is inside the `nginx` namespace, use:

```bash
k get cronjob -n nginx
```

Example:

```text
NAME             SCHEDULE    TIMEZONE   SUSPEND   ACTIVE   LAST SCHEDULE   AGE
cronjob-backup   * * * * *   <none>     False     0        17s             73s
```

---

# 18. Namespace Matters

If you run:

```bash
k get cronjob
```

Kubernetes checks the `default` namespace.

You may get:

```text
No resources found in default namespace.
```

This does not necessarily mean the CronJob doesn't exist.

If it was created in the `nginx` namespace, use:

```bash
k get cronjob -n nginx
```

The same applies to Jobs and Pods:

```bash
k get jobs -n nginx
```

```bash
k get pods -n nginx
```

---

# 19. Watch Jobs Created by the CronJob

Run:

```bash
k get jobs -n nginx
```

Example:

```text
NAME                      STATUS     COMPLETIONS   DURATION   AGE
cronjob-backup-29841499   Complete   1/1           5s         2m
cronjob-backup-29841500   Complete   1/1           5s         1m
cronjob-backup-29841501   Complete   1/1           5s         6s
```

This shows that the CronJob is creating new Jobs.

For example:

```text
cronjob-backup
      │
      ├── cronjob-backup-29841499
      ├── cronjob-backup-29841500
      └── cronjob-backup-29841501
```

Each Job represents one scheduled execution.

---

# 20. Check Pods Created by the CronJob

Each Job creates a Pod.

Run:

```bash
k get pods -n nginx
```

Example:

```text
NAME                                READY   STATUS              RESTARTS   AGE
cronjob-backup-29841499-5jql9       0/1     Completed           0          3m
cronjob-backup-29841500-pnkhp       0/1     Completed           0          2m
cronjob-backup-29841501-vd2gs       0/1     Completed           0          60s
cronjob-backup-29841502-mglrb       0/1     ContainerCreating   0          0s
```

Notice the relationship:

```text
CronJob
   ↓
Job
   ↓
Pod
```

For example:

```text
cronjob-backup
      ↓
cronjob-backup-29841501
      ↓
cronjob-backup-29841501-vd2gs
```

---

# 21. Check Which Node Runs the CronJob Pod

Use:

```bash
k get pods -n nginx -o wide
```

Example:

```text
NAME                                STATUS       NODE
cronjob-backup-29841499-5jql9       Completed    minikube-m03
cronjob-backup-29841500-pnkhp       Completed    minikube-m03
cronjob-backup-29841501-vd2gs       Completed    minikube-m03
```

This shows which Kubernetes Node scheduled the Pods.

---

# 22. CronJob Logs

A CronJob itself does not contain the container logs.

If you try:

```bash
k logs cronjob/cronjob-backup -n nginx
```

you may get:

```text
selector for *v1.CronJob not implemented
```

The reason is that the actual container runs inside a **Pod**.

## Correct Way

First find the Jobs:

```bash
k get jobs -n nginx
```

Then find the Pods:

```bash
k get pods -n nginx
```

Then get the Pod logs:

```bash
k logs <pod-name> -n nginx
```

For example:

```bash
k logs cronjob-backup-29841501-vd2gs -n nginx
```

You can also get logs through a specific Job:

```bash
k logs job/cronjob-backup-29841501 -n nginx
```

---

# 23. Understanding the Backup Command

Our CronJob runs:

```bash
echo Backup Started && mkdir -p /backups && mkdir -p /demo-data && cp -r /demo-data /backups && echo Backup completed
```

The basic flow is:

```text
CronJob
    │
    │ Every minute
    ↓
Job
    │
    ↓
Pod
    │
    ↓
Create backup directory
    │
    ↓
Copy demo data
    │
    ↓
Backup completed
```

After the command finishes:

```text
Pod → Completed
Job → Complete
```

At the next scheduled time:

```text
New Job → New Pod → New Backup
```

---

# 24. Understanding `hostPath`

In the CronJob example we used:

```yaml
volumes:
  - name: data-volume
    hostPath:
      path: /demo-data
      type: DirectoryOrCreate
```

`hostPath` means:

> Use a directory from the Kubernetes Node's filesystem.

For example:

```text
Kubernetes Node
│
├── /demo-data
│
└── /backup
```

The Pod can mount these directories.

---

# 25. Understanding `DirectoryOrCreate`

We used:

```yaml
type: DirectoryOrCreate
```

This means:

> If the directory does not exist on the Node, Kubernetes creates it.

For example:

```text
Node
│
└── /demo-data
```

If `/demo-data` doesn't exist, Kubernetes can create it.

---

# 26. Important `hostPath` Limitation

`hostPath` storage belongs to a specific Node.

For example:

```text
Node 1
└── /demo-data

Node 2
└── /demo-data

Node 3
└── /demo-data
```

These are different directories.

They are **not automatically shared** between Nodes.

Therefore:

```text
hostPath ≠ shared storage
```

For learning and local Minikube practice, `hostPath` is useful.

In production, applications commonly use proper persistent or shared storage solutions instead.

---

# 27. Common CronJob YAML Error

During the practice, we initially received this error:

```text
Error from server (BadRequest): error when creating "cron-job.yml":
CronJob in version "v1" cannot be handled as a CronJob:
strict decoding error:
unknown field "spec.jobTemplate.spec.template.spec.containers[0].volumes"
```

The important part was:

```text
containers[0].volumes
```

This means `volumes` was placed inside the container.

## Incorrect

```yaml
containers:
  - name: backup-container
    image: busybox

    volumes:
      ...
```

`volumes` does not belong inside a container.

## Correct

```yaml
spec:
  containers:
    - name: backup-container
      image: busybox

  volumes:
    - name: data-volume
      hostPath:
        path: /demo-data
        type: DirectoryOrCreate
```

Think of the structure like this:

```text
Pod
├── containers
│   └── Container
│
└── volumes
```

Both `containers` and `volumes` belong directly to the **Pod specification**.

---

# 28. Useful Commands

## Apply a Job

```bash
k apply -f job.yml
```

## Apply a CronJob

```bash
k apply -f cron-job.yml
```

## List Jobs

```bash
k get jobs -n nginx
```

## List CronJobs

```bash
k get cronjob -n nginx
```

## List Pods

```bash
k get pods -n nginx
```

## List Pods with Node Information

```bash
k get pods -n nginx -o wide
```

## Watch Pods

```bash
k get pods -n nginx -w
```

## Get Job Details

```bash
k describe job <job-name> -n nginx
```

## Get CronJob Details

```bash
k describe cronjob <cronjob-name> -n nginx
```

## Get Pod Logs

```bash
k logs <pod-name> -n nginx
```

## Get Job Logs

```bash
k logs job/<job-name> -n nginx
```

## Delete a CronJob

```bash
k delete cronjob <cronjob-name> -n nginx
```

## Delete a Job

```bash
k delete job <job-name> -n nginx
```

## Delete Using YAML

```bash
k delete -f cron-job.yml
```

## Delete a Namespace

```bash
k delete namespace nginx
```

---

# 29. Deleting a Namespace

During the practice, we used:

```bash
k delete ns -n nginx
```

This was incorrect because `-n` specifies the namespace in which a resource exists. It does not provide the name of the namespace to delete.

The correct command is:

```bash
k delete namespace nginx
```

Or:

```bash
k delete ns nginx
```

Output:

```text
namespace "nginx" deleted
```

Deleting a namespace also deletes the resources inside that namespace.

For example:

```text
nginx Namespace
│
├── Pods
├── Jobs
├── CronJobs
├── Deployments
├── Services
└── Other resources
```

When the namespace is deleted, these namespaced resources are removed as well.

---

# 30. Namespace Reminder

If a resource was created with:

```yaml
metadata:
  namespace: nginx
```

you need to specify the namespace when checking it:

```bash
k get jobs -n nginx
```

```bash
k get cronjob -n nginx
```

```bash
k get pods -n nginx
```

Without `-n nginx`, Kubernetes normally checks the `default` namespace.

---

# 31. Complete Mental Model

The most important concepts to remember:

## Job

```text
Job
 │
 └── Pod
      │
      └── Container
           │
           └── Task
```

A Job performs work **one time**.

## CronJob

```text
CronJob
   │
   ├── Job #1
   │    └── Pod #1
   │
   ├── Job #2
   │    └── Pod #2
   │
   ├── Job #3
   │    └── Pod #3
   │
   └── Job #4
        └── Pod #4
```

A CronJob creates new Jobs **according to a schedule**.

---

# 32. Easy Memory Trick

> **Job = Do this work.**

> **CronJob = Do this work again and again on a schedule.**

Remember the complete relationship:

```text
CronJob → Job → Pod → Container → Task
```

And the difference:

```text
Deployment
    ↓
Keep application running


Job
    ↓
Complete task once


CronJob
    ↓
Create Jobs repeatedly according to a schedule
```

---

# 33. Practice Summary

In this practice, we learned:

* What a Kubernetes Job is
* Why Jobs are used
* Difference between Job and Deployment
* How a Job creates a Pod
* How to check Job status
* How to check Job Pods
* How to view Job logs
* What a CronJob is
* Difference between Job and CronJob
* How Cron expressions work
* How a CronJob creates Jobs
* How Jobs create Pods
* How to check CronJobs
* How to check Jobs created by a CronJob
* How to check Pods created by those Jobs
* How to view CronJob Job/Pod logs
* How `jobTemplate` works
* How `hostPath` works
* What `DirectoryOrCreate` means
* Common CronJob YAML errors
* How namespaces affect Kubernetes commands
* How to delete a CronJob
* How to delete a Job
* How to delete a Namespace

## Final Concept

```text
                    Kubernetes
                         │
                         │
                    CronJob
                         │
                  Scheduled time
                         │
                         ↓
                       Job
                         │
                         ↓
                       Pod
                         │
                         ↓
                    Container
                         │
                         ↓
                       Task
                         │
                         ↓
                    Completed
```

> **Job = One-time task**

> **CronJob = Scheduled, repeated Jobs**

