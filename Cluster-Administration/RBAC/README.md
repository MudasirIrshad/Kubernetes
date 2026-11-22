# Kubernetes RBAC and Dashboard --- Beginner-Friendly Complete Guide

> **Goal:** Understand Kubernetes permissions from the very beginning:
> `Role`, `RoleBinding`, `ClusterRole`, `ClusterRoleBinding`,
> ServiceAccounts, permission checks, and how those permissions affect
> the Kubernetes Dashboard.
>
> This guide uses a simple school analogy, practical YAML examples,
> commands for Minikube, and diagrams. It assumes you are learning
> Kubernetes on Linux.

------------------------------------------------------------------------

## 1. First, what problem does RBAC solve?

Imagine a school.

-   Students can enter classrooms and read their books.
-   Teachers can manage students in their classrooms.
-   The principal can manage the whole school.

You would not give every student the principal's keys. Everyone should
only have the permissions they need.

A Kubernetes cluster has the same problem. Many people and applications
may need to interact with it, but they should not all be allowed to do
everything.

**RBAC** stands for **Role-Based Access Control**. It is Kubernetes'
permission system. It answers three questions:

1.  **Who** is making the request?
2.  **What** action are they trying to perform?
3.  **Which** resource are they trying to access, and where?

For example:

-   Who: `apache-user` ServiceAccount
-   Action: `get`
-   Resource: `pods`
-   Namespace: `apache`

RBAC decides whether that request is allowed.

### A simple picture

``` text
             A request reaches Kubernetes API Server
                            |
                            v
                  Who is making the request?
                            |
                            v
                  What are they requesting?
                   (verb + resource + scope)
                            |
                            v
                    RBAC checks permissions
                       /             \
                      v               v
                  Allowed           Denied
                      |               |
                      v               v
              Request proceeds    Forbidden error
```

The API server is the front desk of the cluster. `kubectl`,
applications, and the Dashboard send requests to it. RBAC is one of the
checks the API server performs.

------------------------------------------------------------------------

## 2. Understand the four important RBAC objects

There are four main objects:

  -----------------------------------------------------------------------
  Object                              Simple meaning
  ----------------------------------- -----------------------------------
  `Role`                              A permission list for resources in
                                      one namespace

  `RoleBinding`                       Connects a person or ServiceAccount
                                      to a Role in one namespace

  `ClusterRole`                       A reusable permission list that can
                                      cover namespaced or cluster-wide
                                      resources

  `ClusterRoleBinding`                Connects an identity to a
                                      ClusterRole across the cluster
  -----------------------------------------------------------------------

Remember this distinction:

-   A **Role or ClusterRole defines permissions**.
-   A **RoleBinding or ClusterRoleBinding assigns permissions to an
    identity**.

A Role by itself does not give anybody permission. A binding is needed
to connect the permissions to a subject.

### The school analogy

  -----------------------------------------------------------------------
  Kubernetes                          School analogy
  ----------------------------------- -----------------------------------
  Subject (User or ServiceAccount)    Student or teacher

  Role / ClusterRole                  Written list of allowed activities

  RoleBinding / ClusterRoleBinding    The official assignment of that
                                      list to a person

  Namespace                           A classroom or school department

  Cluster                             The entire school
  -----------------------------------------------------------------------

------------------------------------------------------------------------

## 3. What is a namespace?

A namespace is a way to separate resources inside one Kubernetes
cluster.

Imagine one school with different classrooms:

``` text
                 Kubernetes Cluster
                         |
          +--------------+--------------+
          |              |              |
       apache           nginx         database
      namespace        namespace      namespace
          |              |              |
       Pods           Deployments       Pods
       Services       Services          Services
       Deployments    Pods              Secrets
```

The cluster is still one cluster. Namespaces provide logical separation.

See your namespaces:

``` bash
kubectl get namespaces
```

Or, because you use the `k` alias:

``` bash
k get ns
```

See Pods in a particular namespace:

``` bash
k get pods -n apache
```

If you omit `-n`, kubectl normally uses the namespace configured in your
current context, often `default`.

**Important:** A namespace is not a separate cluster and is not, by
itself, a complete security boundary. RBAC can use namespaces to limit
access to namespaced resources.

------------------------------------------------------------------------

## 4. What is a subject?

A subject is the identity receiving permissions. In RBAC, subjects can
be:

-   A normal Kubernetes user
-   A group
-   A ServiceAccount

### What is a ServiceAccount?

A ServiceAccount is an identity intended mainly for applications and
processes running in Kubernetes.

Imagine an application is a worker in your school. It needs an identity
card to prove who it is when it talks to the Kubernetes API.

You created this ServiceAccount:

**`service-account.yaml`**

``` yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: apache-user
  namespace: apache
```

Apply it:

``` bash
k apply -f service-account.yaml
```

Check it:

``` bash
k get serviceaccounts -n apache
```

You should see `apache-user` as well as the namespace's `default`
ServiceAccount.

### Why does the ServiceAccount have a namespace?

A ServiceAccount is a namespaced resource. The identity is made up of
its namespace and name.

For example:

``` text
system:serviceaccount:apache:apache-user
                 |       |
                 |       +-- ServiceAccount name
                 +---------- Namespace
```

The full username used by Kubernetes for this ServiceAccount is:

``` text
system:serviceaccount:apache:apache-user
```

### What about normal users?

Kubernetes does not normally store human users as regular API objects in
the way it stores ServiceAccounts. Human identities are commonly
provided by an external authentication system, certificates, or an
identity provider.

A ServiceAccount is therefore not the same thing as a human login
account.

------------------------------------------------------------------------

## 5. Understand API groups, resources, and verbs

A Role's rules describe what actions are allowed.

A rule commonly contains:

``` yaml
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
```

Think of it as:

> In this API group, allow these actions on these resources.

### 5.1 `apiGroups`

An API group identifies which part of the Kubernetes API owns a
resource.

Some common examples:

  Resource                 API group
  ------------------------ -----------------------------
  Pods                     `""` (core API group)
  Services                 `""`
  ConfigMaps               `""`
  Secrets                  `""`
  Deployments              `apps`
  ReplicaSets              `apps`
  StatefulSets             `apps`
  Jobs                     `batch`
  CronJobs                 `batch`
  Roles and RoleBindings   `rbac.authorization.k8s.io`

The empty string `""` is important. It means the **core API group**. It
is not the same as writing `apiVersion: v1` in a manifest.

### 5.2 `resources`

Resources are the Kubernetes objects you want to control. They are
normally written as lowercase plural names:

``` yaml
resources: ["pods", "services", "deployments"]
```

Examples include:

-   `pods`
-   `services`
-   `deployments`
-   `configmaps`
-   `secrets`
-   `jobs`

### 5.3 `verbs`

Verbs are the actions someone is allowed to perform.

  Verb                 Simple meaning
  -------------------- -------------------------------------------------
  `get`                Read one named object
  `list`               List objects
  `watch`              Watch for changes
  `create`             Create a new object
  `update`             Replace/update an object
  `patch`              Make a partial change
  `delete`             Delete an object
  `deletecollection`   Delete a collection of objects, where supported

For example:

``` yaml
verbs: ["get", "list", "watch"]
```

means read-only access for the specified resources.

**Important:** `apply` is not an RBAC verb. When you run
`kubectl apply`, kubectl uses API operations such as create or patch,
depending on the situation.

------------------------------------------------------------------------

## 6. Role: permissions inside one namespace

A `Role` defines permissions within one namespace.

Imagine you have a classroom called `apache`. You want `apache-user` to
manage Pods, Services, and Deployments in that classroom, but not in
every other classroom.

Create this file:

**`role.yaml`**

``` yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: apache-manager
  namespace: apache
rules:
  - apiGroups: ["", "apps"]
    resources: ["pods", "services", "deployments"]
    verbs:
      - get
      - list
      - watch
      - create
      - update
      - patch
      - delete
```

Apply it:

``` bash
k apply -f role.yaml
```

Check it:

``` bash
k get roles -n apache
```

Inspect its details:

``` bash
k describe role apache-manager -n apache
```

### Read the YAML line by line

-   `apiVersion`: Which Kubernetes API version defines this object.
-   `kind: Role`: We are creating a namespace-scoped Role.
-   `metadata.name`: The Role's name is `apache-manager`.
-   `metadata.namespace`: The Role belongs to `apache`.
-   `rules`: The permissions written in this Role.
-   `apiGroups: ["", "apps"]`: It covers the core API group and the
    `apps` API group.
-   `resources`: It covers Pods, Services, and Deployments.
-   `verbs`: It allows reading, creating, changing, and deleting those
    resources.

**A useful correction:** Do not add an API group unless you also specify
resources from that group. For example, adding `batch` while listing
only Pods, Services, and Deployments does not grant Job or CronJob
permissions. To manage Jobs, include resources such as `jobs` and the
`batch` API group.

### What this Role does not allow

It does not automatically allow the subject to:

-   Manage resources in the `nginx` namespace.
-   Manage Nodes or PersistentVolumes.
-   Manage every resource in the cluster.
-   Create or edit RBAC permissions unless those permissions are
    explicitly included.

A Role is limited to its namespace for namespaced resources.

------------------------------------------------------------------------

## 7. RoleBinding: assign a Role to a ServiceAccount

You have created:

1.  ServiceAccount: `apache-user`
2.  Role: `apache-manager`

But the ServiceAccount still does not receive the Role's permissions
until you bind them.

A RoleBinding is like handing the classroom permission list to a
particular student or teacher.

Create:

**`role-binding.yaml`**

``` yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: apache-binding
  namespace: apache
subjects:
  - kind: ServiceAccount
    name: apache-user
    namespace: apache
roleRef:
  kind: Role
  name: apache-manager
  apiGroup: rbac.authorization.k8s.io
```

Apply:

``` bash
k apply -f role-binding.yaml
```

Check:

``` bash
k get rolebindings -n apache
```

Inspect:

``` bash
k describe rolebinding apache-binding -n apache
```

### Understand `subjects`

``` yaml
subjects:
  - kind: ServiceAccount
    name: apache-user
    namespace: apache
```

This says:

> Give the permissions to the ServiceAccount named `apache-user` in the
> `apache` namespace.

### Understand `roleRef`

``` yaml
roleRef:
  kind: Role
  name: apache-manager
  apiGroup: rbac.authorization.k8s.io
```

This says:

> The permission list to assign is the Role named `apache-manager`.

Notice that the API group here is `rbac.authorization.k8s.io`, because
Role is an RBAC object.

### Diagram: Role + RoleBinding

``` text
       ServiceAccount                 Role
         apache-user             apache-manager
              |                         |
              |                         |
              +-----------+-------------+
                          |
                          v
                     RoleBinding
                   apache-binding
                          |
                          v
             Permissions in namespace apache
```

The Role defines the permissions. The RoleBinding connects those
permissions to the ServiceAccount.

### Important field difference

In a ServiceAccount subject, `apiGroup` is not
`rbac.authorization.k8s.io`. The ServiceAccount is in the core API
group. For a ServiceAccount subject, omit `apiGroup` or set it to an
empty string if your manifest includes it.

By contrast, `roleRef.apiGroup` should be:

``` yaml
apiGroup: rbac.authorization.k8s.io
```

These two fields have different meanings.

------------------------------------------------------------------------

## 8. Test permissions with `kubectl auth can-i`

Kubernetes provides a useful command to ask:

> Is this identity allowed to perform this action on this resource?

The basic syntax is:

``` bash
kubectl auth can-i VERB RESOURCE
```

Examples for your current logged-in identity:

``` bash
k auth can-i get pods -n apache
k auth can-i create deployments -n apache
k auth can-i delete services -n apache
```

List permissions for the current identity in a namespace:

``` bash
k auth can-i --list -n apache
```

### Test the ServiceAccount specifically

When impersonating a ServiceAccount, use its full username:

``` bash
k auth can-i get pods -n apache \
  --as=system:serviceaccount:apache:apache-user
```

Try creating a Pod:

``` bash
k auth can-i create pods -n apache \
  --as=system:serviceaccount:apache:apache-user
```

Because `apache-manager` includes `get` and `create` for Pods, both
checks should return `yes`, assuming the RoleBinding was applied
correctly and your current account is allowed to impersonate that
ServiceAccount.

### A common beginner mistake

This is not the correct way to impersonate a ServiceAccount:

``` bash
k auth can-i get pods -n apache --as=apache-user
```

That asks Kubernetes to impersonate a normal user named `apache-user`.
It does not identify the ServiceAccount in the `apache` namespace.

Use:

``` text
system:serviceaccount:NAMESPACE:SERVICEACCOUNT_NAME
```

For your example:

``` text
system:serviceaccount:apache:apache-user
```

### What does `whoami` do?

``` bash
k auth whoami
```

It reports the identity Kubernetes sees for your current credentials. It
does not automatically tell you which permissions that identity has. Use
`can-i` to test permissions.

**Note:** Impersonation checks require the identity running the command
to have permission to impersonate the target identity.

------------------------------------------------------------------------

## 9. ClusterRole: reusable permissions that can cover the cluster

Now imagine the principal wants to create a permission list that can be
reused across different classrooms---or that includes cluster-level
resources.

That is where `ClusterRole` comes in.

A `ClusterRole` is a cluster-scoped RBAC object. Unlike a Role, it does
not have a namespace in its metadata.

A ClusterRole can define permissions for:

1.  Namespaced resources, such as Pods and Deployments.
2.  Cluster-scoped resources, such as Nodes and PersistentVolumes.
3.  Non-resource URL paths, when configured.

Example:

**`cluster-role.yaml`**

``` yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-observer
rules:
  - apiGroups: [""]
    resources: ["pods", "services", "namespaces", "nodes"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["apps"]
    resources: ["deployments", "replicasets"]
    verbs: ["get", "list", "watch"]
```

Apply:

``` bash
k apply -f cluster-role.yaml
```

Check:

``` bash
k get clusterroles
k describe clusterrole cluster-observer
```

This ClusterRole defines read permissions for the listed resources.

### Role versus ClusterRole

  -----------------------------------------------------------------------
  Feature                 Role                    ClusterRole
  ----------------------- ----------------------- -----------------------
  Is the RBAC object      Yes                     No
  namespaced?                                     

  Can define permissions  Yes                     Yes
  for Pods?                                       

  Can define permissions  No                      Yes
  for Nodes?                                      

  Can be assigned with    Yes                     Yes
  RoleBinding?                                    

  Can be assigned with    No                      Yes
  ClusterRoleBinding?                             

  Scope when assigned     One namespace           One namespace for
  with RoleBinding                                namespaced resources

  Scope when assigned     Not applicable          Cluster-wide
  with ClusterRoleBinding                         
  -----------------------------------------------------------------------

### A very important concept: ClusterRole + RoleBinding

A ClusterRole does not always mean cluster-wide access.

You can bind a ClusterRole using a **RoleBinding** in one namespace. The
namespaced-resource permissions are then limited to that namespace.

For example:

``` text
                 ClusterRole
               cluster-observer
                       |
                       |
                 RoleBinding
                in namespace apache
                       |
                       v
          Namespaced resources in apache
```

This is useful when you want one reusable permission definition but want
to assign it separately in different namespaces.

A ClusterRole that includes cluster-scoped resources such as Nodes
cannot make those resources accessible through a namespaced RoleBinding.
Access to cluster-scoped resources requires a ClusterRoleBinding.

------------------------------------------------------------------------

## 10. ClusterRoleBinding: assign a ClusterRole across the cluster

A `ClusterRoleBinding` connects a subject to a ClusterRole across the
cluster.

Imagine the principal gives someone a permission card that works
throughout the entire school, not just one classroom.

### Your Dashboard ServiceAccount

You created this ServiceAccount:

**`dashboard-admin-user.yaml`**

``` yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
```

Apply it:

``` bash
k apply -f dashboard-admin-user.yaml
```

Check it:

``` bash
k get serviceaccounts -n kubernetes-dashboard
```

The ServiceAccount belongs to the `kubernetes-dashboard` namespace.

### Bind it to the built-in `cluster-admin` ClusterRole

Your ClusterRoleBinding should look like this:

**`clusterRoleBinding.yaml`**

``` yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user-binding
subjects:
  - kind: ServiceAccount
    name: admin-user
    namespace: kubernetes-dashboard
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
```

Apply:

``` bash
k apply -f clusterRoleBinding.yaml
```

Verify:

``` bash
k get clusterrolebinding admin-user-binding
k describe clusterrolebinding admin-user-binding
```

### Two important corrections to the original YAML

**Correction 1 --- `subjects[].apiGroup`**

You originally wrote:

``` yaml
subjects:
  - kind: ServiceAccount
    name: admin-user
    apiGroup: rbac.authorization.k8s.io
    namespace: kubernetes-dashboard
```

That caused this error:

``` text
subjects[0].apiGroup: Unsupported value: "rbac.authorization.k8s.io"
```

A ServiceAccount is in the core API group. For a ServiceAccount subject,
omit `apiGroup` or use an empty string. Do not put
`rbac.authorization.k8s.io` there.

**Correction 2 --- `metadata.namespace`**

You originally included:

``` yaml
metadata:
  name: admin-user-binding
  namespace: kubernetes-dashboard
```

Remove `metadata.namespace`. A ClusterRoleBinding is cluster-scoped. It
does not belong to a namespace.

The ServiceAccount subject still needs its namespace:

``` yaml
subjects:
  - kind: ServiceAccount
    name: admin-user
    namespace: kubernetes-dashboard
```

### What is `cluster-admin`?

`cluster-admin` is a built-in ClusterRole with extremely broad
administrative permissions.

When you bind it using a ClusterRoleBinding, the subject receives broad
access across the cluster, including resources in all namespaces and
cluster-scoped resources, subject to Kubernetes authorization and
admission controls.

### Diagram: ClusterRoleBinding

``` text
        ServiceAccount
          admin-user
              |
              |
              v
       ClusterRoleBinding
       admin-user-binding
              |
              |
              v
        ClusterRole
        cluster-admin
              |
              v
      Broad cluster-wide access
      +-----------------------+
      | apache namespace      |
      | nginx namespace       |
      | default namespace     |
      | other namespaces      |
      | Nodes / PVs / etc.    |
      +-----------------------+
```

**Warning:** This is full administrative access. It is acceptable as a
learning experiment in a disposable Minikube cluster, but it is not a
safe default for a production Dashboard login.

### Test the binding

``` bash
k auth can-i get pods -n apache \
  --as=system:serviceaccount:kubernetes-dashboard:admin-user
```

Try a cluster-scoped permission:

``` bash
k auth can-i list nodes \
  --as=system:serviceaccount:kubernetes-dashboard:admin-user
```

Both should return `yes` when the binding is correctly applied and your
current identity can impersonate the ServiceAccount.

------------------------------------------------------------------------

## 11. RoleBinding versus ClusterRoleBinding --- the key difference

This is one of the most important things to understand.

### RoleBinding

A RoleBinding grants permissions within the namespace where the
RoleBinding exists. It can reference either:

-   A Role in that namespace, or
-   A ClusterRole, with its namespaced-resource permissions scoped to
    that namespace.

``` text
                  RoleBinding
                 namespace apache
                       |
                       v
              Permissions in apache
```

### ClusterRoleBinding

A ClusterRoleBinding grants the referenced ClusterRole's permissions
cluster-wide.

``` text
                ClusterRoleBinding
                       |
                       v
               Cluster-wide access
                       |
          +------------+------------+
          |            |            |
        apache       nginx        default
```

### Quick memory trick

-   `Role` = permission list for one namespace.
-   `RoleBinding` = assign permissions in one namespace.
-   `ClusterRole` = cluster-level/reusable permission definition.
-   `ClusterRoleBinding` = assign ClusterRole permissions cluster-wide.

------------------------------------------------------------------------

## 12. Kubernetes Dashboard: GUI for your cluster

The Kubernetes Dashboard is a web-based user interface for viewing and
managing Kubernetes resources.

You started it with:

``` bash
minikube dashboard
```

Minikube checks the Dashboard, starts a local proxy, and opens a browser
URL similar to:

``` text
http://127.0.0.1:<port>/api/v1/namespaces/kubernetes-dashboard/services/http:kubernetes-dashboard:/proxy/
```

### What does `127.0.0.1` mean?

`127.0.0.1` means your own computer, also called localhost.

The port is a local port opened by the Minikube proxy. It allows your
browser to reach the Dashboard service through the Kubernetes API proxy.

### Is Dashboard only for looking?

No. Dashboard can support both viewing and managing resources, depending
on the Dashboard version and the permissions of the identity used for
its API requests.

Examples of actions that a Dashboard interface may provide:

-   View Pods, Deployments, Services, and namespaces.
-   Inspect resource details and status.
-   View logs and events.
-   Create resources.
-   Edit or update resources.
-   Delete resources.
-   Scale Deployments.

The exact screens and actions vary by Dashboard version and supported
resource type.

### The most important idea

**Opening the Dashboard website does not automatically give you
administrative permissions.**

Think of it like this:

``` text
        You open the Dashboard URL
                   |
                   v
          Can your browser reach it?
                   |
                   v
            Dashboard loads
                   |
                   v
       An API request is made to Kubernetes
                   |
                   v
        Kubernetes authenticates identity
                   |
                   v
          RBAC checks that identity
                   |
             +-----+-----+
             |           |
             v           v
          Allowed      Forbidden
             |           |
             v           v
       Action succeeds  Action denied
```

The URL only proves that the browser can reach the Dashboard route. It
does not prove that the identity has permission to create, edit, or
delete resources.

### Can you manage resources without `cluster-admin`?

Yes.

You do **not** need `cluster-admin` to manage resources through a GUI.
You need the relevant permissions for the actions you want to perform.

For example, an identity with these permissions in the `apache`
namespace can manage the specified resources there:

``` yaml
rules:
  - apiGroups: [""]
    resources: ["pods", "services"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```

That identity should not automatically be able to manage resources in
`nginx` or `default`, nor should it be able to manage Nodes.

### Your question: why create `dashboard-admin-user`?

You do not need to create it merely to open the Dashboard URL.

You create it when you want a separate, explicit ServiceAccount identity
for Dashboard authentication. Then you bind permissions to it.

For learning, your setup is:

``` text
ServiceAccount: admin-user
Namespace: kubernetes-dashboard
             |
             v
ClusterRoleBinding: admin-user-binding
             |
             v
ClusterRole: cluster-admin
             |
             v
Broad administrative permissions
```

This makes the ServiceAccount highly privileged. It does not mean every
Dashboard installation automatically uses that account. You must
authenticate to the Dashboard using the intended identity through the
authentication method supported by your Dashboard version.

**Important distinction:** Creating the ServiceAccount and binding it
does not, by itself, log you into the Dashboard as that account.

------------------------------------------------------------------------

## 13. Getting a ServiceAccount token for Dashboard login

For Dashboard versions that support bearer-token login, you can request
a token for the ServiceAccount.

On modern Kubernetes versions, use:

``` bash
k create token admin-user -n kubernetes-dashboard
```

This prints a token. Treat it like a password:

-   Do not share it in screenshots or chat.
-   Do not commit it to Git.
-   Do not save it in a public notes repository.
-   Do not leave it lying around on a shared computer.

Copy the token into the Dashboard's token login screen if your installed
Dashboard version uses that login flow.

### Important version note

Dashboard installation, authentication, and login behavior can vary by
version. Follow the documentation for the exact version installed in
your cluster. Do not assume that simply opening the Minikube proxy URL
logs you in as `admin-user`.

Also, do not create a long-lived token Secret just because an old
tutorial tells you to. Modern Kubernetes supports short-lived
ServiceAccount tokens through `kubectl create token`.

------------------------------------------------------------------------

## 14. How to check your Dashboard ServiceAccount's permissions

You can test the ServiceAccount without logging into the Dashboard.

Check whether it can list Pods in `apache`:

``` bash
k auth can-i list pods -n apache \
  --as=system:serviceaccount:kubernetes-dashboard:admin-user
```

Check whether it can delete Deployments in `apache`:

``` bash
k auth can-i delete deployments.apps -n apache \
  --as=system:serviceaccount:kubernetes-dashboard:admin-user
```

Check whether it can list Nodes:

``` bash
k auth can-i list nodes \
  --as=system:serviceaccount:kubernetes-dashboard:admin-user
```

With the `cluster-admin` ClusterRoleBinding, these checks should
normally return `yes`.

If a check returns `no`, inspect:

``` bash
k get clusterrolebinding admin-user-binding -o yaml
k get serviceaccount admin-user -n kubernetes-dashboard
k describe clusterrole cluster-admin
```

Remember that `--as` impersonation itself requires permission from the
identity running the command.

------------------------------------------------------------------------

## 15. Useful commands to remember

### Inspect RBAC objects

``` bash
k get roles -A
k get rolebindings -A
k get clusterroles
k get clusterrolebindings
```

### Inspect a specific Role and RoleBinding

``` bash
k describe role apache-manager -n apache
k describe rolebinding apache-binding -n apache
```

### Inspect a specific ClusterRole and ClusterRoleBinding

``` bash
k describe clusterrole cluster-admin
k describe clusterrolebinding admin-user-binding
```

### Check current identity

``` bash
k auth whoami
```

### Check permissions

``` bash
k auth can-i get pods -n apache
k auth can-i create deployments -n apache
k auth can-i --list -n apache
```

### Test a ServiceAccount

``` bash
k auth can-i list pods -n apache \
  --as=system:serviceaccount:apache:apache-user
```

### Check Dashboard resources

``` bash
k get pods -n kubernetes-dashboard
k get services -n kubernetes-dashboard
k get serviceaccounts -n kubernetes-dashboard
```

### See API resource names and API groups

``` bash
k api-resources
```

For example, this helps you discover the full resource name and API
group for resources such as Vertical Pod Autoscalers.

------------------------------------------------------------------------

## 16. Common mistakes and how to understand them

### Mistake 1: Using `version` instead of `apiVersion`

Incorrect:

``` yaml
version: rbac.authorization.k8s.io/v1
```

Correct:

``` yaml
apiVersion: rbac.authorization.k8s.io/v1
```

Kubernetes manifests use `apiVersion`.

### Mistake 2: Using `apiGroup` incorrectly for a ServiceAccount subject

Incorrect:

``` yaml
subjects:
  - kind: ServiceAccount
    name: admin-user
    apiGroup: rbac.authorization.k8s.io
    namespace: kubernetes-dashboard
```

Correct:

``` yaml
subjects:
  - kind: ServiceAccount
    name: admin-user
    namespace: kubernetes-dashboard
```

The `apiGroup` field is omitted for a ServiceAccount subject.

### Mistake 3: Adding a namespace to a ClusterRoleBinding

Incorrect:

``` yaml
metadata:
  name: admin-user-binding
  namespace: kubernetes-dashboard
```

Correct:

``` yaml
metadata:
  name: admin-user-binding
```

ClusterRoleBindings are cluster-scoped.

### Mistake 4: Confusing a Role with a RoleBinding

A Role only defines permissions. It does not assign those permissions to
anybody.

You need a binding to connect the Role to a subject.

### Mistake 5: Thinking a ClusterRole always grants cluster-wide access

A ClusterRole is a cluster-scoped permission definition. If you attach
it through a RoleBinding, namespaced-resource access is scoped to that
RoleBinding's namespace.

A ClusterRoleBinding is what assigns those permissions cluster-wide.

### Mistake 6: Thinking `kubectl apply` is a verb

`apply` is a kubectl command, not an RBAC verb. RBAC rules use verbs
such as `create`, `get`, `patch`, and `update`.

### Mistake 7: Assuming the Dashboard URL means you are an administrator

Opening the URL only means the web route is reachable. Authentication
and RBAC authorization are separate checks.

### Mistake 8: Giving `cluster-admin` to every ServiceAccount

This gives far more power than most applications need. Use the smallest
set of permissions required.

------------------------------------------------------------------------

## 17. A practical Minikube learning exercise

Use this exercise to connect all the concepts.

### Part A --- Create the namespace

``` bash
k create namespace apache
```

### Part B --- Create the ServiceAccount

Create `service-account.yaml`:

``` yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: apache-user
  namespace: apache
```

Apply:

``` bash
k apply -f service-account.yaml
```

### Part C --- Create the Role

Create `role.yaml`:

``` yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: apache-manager
  namespace: apache
rules:
  - apiGroups: [""]
    resources: ["pods", "services"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```

Apply:

``` bash
k apply -f role.yaml
```

### Part D --- Bind the Role

Create `role-binding.yaml`:

``` yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: apache-binding
  namespace: apache
subjects:
  - kind: ServiceAccount
    name: apache-user
    namespace: apache
roleRef:
  kind: Role
  name: apache-manager
  apiGroup: rbac.authorization.k8s.io
```

Apply:

``` bash
k apply -f role-binding.yaml
```

### Part E --- Test the permissions

``` bash
k auth can-i list pods -n apache \
  --as=system:serviceaccount:apache:apache-user
```

Expected:

``` text
yes
```

Now test access to another namespace:

``` bash
k auth can-i list pods -n default \
  --as=system:serviceaccount:apache:apache-user
```

Expected, assuming no other binding grants access:

``` text
no
```

Try managing Nodes:

``` bash
k auth can-i list nodes \
  --as=system:serviceaccount:apache:apache-user
```

Expected, assuming no other binding grants access:

``` text
no
```

This exercise proves the key idea: your ServiceAccount can manage the
resources allowed by its Role in `apache`, but it does not automatically
get access to other namespaces or cluster-scoped resources.

------------------------------------------------------------------------

## 18. Security: why `cluster-admin` must be handled carefully

The built-in `cluster-admin` ClusterRole is extremely powerful. A
subject with this permission can potentially:

-   Read sensitive Secrets.
-   Create or delete workloads.
-   Change important cluster resources.
-   Change RBAC permissions and grant access to others.
-   Disrupt applications and services.
-   Access resources across namespaces.

A compromised token for a `cluster-admin` ServiceAccount can therefore
compromise the whole cluster.

### Safer production approach

For a real cluster:

1.  Avoid using `cluster-admin` for routine Dashboard access.
2.  Create a dedicated ServiceAccount or use an appropriate human
    identity provider.
3.  Grant only the verbs and resources required.
4.  Use a RoleBinding if access should be limited to one namespace.
5.  Use short-lived credentials where possible.
6.  Protect tokens like passwords.
7.  Review and remove unused bindings.

For a disposable local Minikube learning cluster, experimenting with
`cluster-admin` can help you understand the full capabilities. Do not
copy that setup into production without a security review.

------------------------------------------------------------------------

## 19. One-page revision

``` text
RBAC = Kubernetes permission system

SUBJECT = Who receives permission?
  - User
  - Group
  - ServiceAccount

ROLE = What can they do in one namespace?
ROLEBINDING = Assign a Role in one namespace.

CLUSTERROLE = Reusable/cluster-scoped permission definition.
CLUSTERROLEBINDING = Assign a ClusterRole across the cluster.

RULE =
  apiGroups = Which API group?
  resources = Which Kubernetes objects?
  verbs = Which actions?

DASHBOARD =
  GUI to view and manage resources.
  Actions depend on the authenticated identity's RBAC permissions.

OPENING DASHBOARD URL != CLUSTER ADMIN ACCESS
```

### Final school analogy

``` text
Kubernetes Cluster                  School
------------------                  ------
Cluster                              Entire school
Namespace                            Classroom
Pod                                  Student's desk
Service                              School communication point
ServiceAccount                       Application identity card
Role                                 Classroom permission list
RoleBinding                          Assign permission list in a classroom
ClusterRole                          Reusable/cluster-level permission list
ClusterRoleBinding                   Assign permissions across the school
Kubernetes API Server                School front office
RBAC                                 Rules checked by the front office
Dashboard                            School management website
```

If you remember only one thing, remember this:

**A Role or ClusterRole describes what is allowed. A RoleBinding or
ClusterRoleBinding decides who receives those permissions. The Dashboard
is simply a GUI; Kubernetes RBAC still decides which actions that
identity can perform.**

------------------------------------------------------------------------

## 20. Official documentation

Use official Kubernetes documentation when checking details for your
installed version:

-   [Kubernetes RBAC
    authorization](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
-   [Kubernetes Service
    Accounts](https://kubernetes.io/docs/concepts/security/service-accounts/)
-   [Kubernetes
    authentication](https://kubernetes.io/docs/reference/access-authn-authz/authentication/)
-   [Kubernetes Dashboard
    documentation](https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/)

**Dashboard lifecycle note:** Kubernetes Dashboard is deprecated and is
no longer actively maintained. For a new long-term cluster management
UI, review currently maintained alternatives such as Headlamp. The RBAC
concepts in this guide still apply regardless of which GUI you use.
