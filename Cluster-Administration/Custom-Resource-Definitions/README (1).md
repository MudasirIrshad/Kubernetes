# Kubernetes Custom Resource Definitions (CRDs)

A beginner-friendly guide using the `DevOpsBatch` resource.

## 1. What problem does a CRD solve?

Kubernetes comes with built-in resource types, including:

-   **Pod** --- runs one or more containers.
-   **Deployment** --- manages replicated Pods and rolling updates.
-   **Service** --- provides a stable way to reach Pods.
-   **ConfigMap** --- stores non-sensitive configuration.
-   **Secret** --- stores sensitive configuration data.

Kubernetes knows what these resources mean because their types are built
into its API.

Imagine that you are building a platform to manage DevOps training
batches. You want each batch to have:

-   A name
-   A duration
-   A learning mode
-   A platform

Kubernetes does not have a built-in resource type called `DevOpsBatch`.
If you try this before defining it:

``` bash
kubectl get devopsbatches
```

Kubernetes will not recognize that resource type.

A **Custom Resource Definition (CRD)** solves this problem. It extends
the Kubernetes API by registering a new resource type. After the CRD is
installed, Kubernetes can accept, validate, store, and return objects of
that type.

In simple words:

> A CRD teaches Kubernetes the structure and name of a new kind of
> resource.

A CRD does not, by itself, make Kubernetes perform special work when
that resource is created. Automation usually requires a controller or
Operator, explained later in this guide.

## 2. The school example

Think of Kubernetes as a school management system.

The system already knows about students, classrooms, and teachers. But
your school introduces a new concept: a `DevOpsBatch`.

You first define what a DevOpsBatch looks like. That definition is the
**CRD**.

Then you create a particular batch, such as `junoon`. That object is a
**Custom Resource (CR)**.

The distinction is:

  -----------------------------------------------------------------------
  Term                    Meaning                 Example
  ----------------------- ----------------------- -----------------------
  CRD                     Defines a resource type The definition of
                                                  `DevOpsBatch`

  Custom Resource (CR)    An instance of that     The `junoon` batch
                          type                    

  Controller / Operator   Watches resources and   Automatically creates
                          takes action            or manages something
                                                  for a batch
  -----------------------------------------------------------------------

Another useful comparison:

-   **CRD = blueprint**
-   **CR = one object built from the blueprint**
-   **Controller = worker that acts on the object**

## 3. Your DevOpsBatch CRD

The following is a corrected and beginner-friendly version of your CRD.
It uses the conventional Kubernetes structure: custom configuration
fields are placed under `spec`.

Save this as `devopsbatch-crd.yaml`:

``` yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition

metadata:
  name: devopsbatches.devops.mudasirirshad.com

spec:
  group: devops.mudasirirshad.com

  names:
    plural: devopsbatches
    singular: devopsbatch
    kind: DevOpsBatch
    shortNames:
      - junoon
      - batches
      - tws

  scope: Namespaced

  versions:
    - name: v1
      served: true
      storage: true

      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              description: "Configuration for a DevOps training batch"
              properties:
                name:
                  type: string
                  description: "Name of the DevOps batch"

                duration:
                  type: string
                  description: "Duration of the DevOps batch"

                mode:
                  type: string
                  description: "Learning mode, such as Online or Onsite"

                platform:
                  type: string
                  description: "Platform used by the batch"
```

### 3.1 Understand each part

#### `apiVersion`

``` yaml
apiVersion: apiextensions.k8s.io/v1
```

This is the API version for defining a CRD itself. It tells Kubernetes
which version of the CRD API format this YAML uses.

Do not confuse it with the API version of the DevOpsBatch resources that
you will create later.

#### `kind`

``` yaml
kind: CustomResourceDefinition
```

This says that the object in this file is a CRD definition. You are
registering a new resource type, not creating a batch yet.

#### `metadata.name`

``` yaml
metadata:
  name: devopsbatches.devops.mudasirirshad.com
```

The CRD name follows this pattern:

``` text
<plural>.<group>
```

For this example:

-   `devopsbatches` is the plural resource name.
-   `devops.mudasirirshad.com` is the API group.

The CRD's metadata name must match the plural and group declared in its
specification.

#### `spec.group`

``` yaml
group: devops.mudasirirshad.com
```

An API group organizes related Kubernetes resource types. Your
DevOpsBatch resource will use this group.

#### `spec.names`

``` yaml
names:
  plural: devopsbatches
  singular: devopsbatch
  kind: DevOpsBatch
  shortNames:
    - junoon
    - batches
    - tws
```

These names tell Kubernetes how the resource is identified.

  -----------------------------------------------------------------------
  Field                               Purpose
  ----------------------------------- -----------------------------------
  `plural`                            Main collection name, used by
                                      `kubectl get devopsbatches`

  `singular`                          Singular name, used by commands
                                      such as `kubectl get devopsbatch`

  `kind`                              Kind written in a Custom Resource
                                      YAML

  `shortNames`                        Optional aliases for `kubectl`
  -----------------------------------------------------------------------

For example, `kubectl get batches` and `kubectl get tws` are aliases for
listing the same resource type. Short names are aliases, not separate
resource types. These particular aliases are just examples; choose names
that are clear and do not conflict with other resource aliases.

#### `spec.scope`

``` yaml
scope: Namespaced
```

This means each DevOpsBatch belongs to a Kubernetes namespace.

For example, you can create one batch in the `development` namespace and
another in the `production` namespace. They are separate namespaced
objects.

The other possible scope is `Cluster`, which means the custom resources
are not namespaced.

#### `spec.versions`

``` yaml
versions:
  - name: v1
    served: true
    storage: true
```

This defines the API version for your custom resource.

-   `name: v1` --- the version name.
-   `served: true` --- Kubernetes makes this version available through
    its API.
-   `storage: true` --- Kubernetes stores the resource using this
    version.

A CRD must have exactly one version with `storage: true`.

#### `openAPIV3Schema`

This section describes the shape of a valid DevOpsBatch object.
Kubernetes uses this schema to validate submitted resources.

For example:

``` yaml
spec:
  name: Junoon
  duration: 6 months
  mode: Online
  platform: Kubernetes
```

The schema says that these four values must be strings.

### 3.2 Why put fields under `spec`?

Kubernetes resources commonly separate their desired configuration from
their observed state:

-   `spec` describes what you want.
-   `status` is commonly used by a controller to report what it has
    observed or accomplished.

For example, `spec` might say that a batch should run for six months. A
future controller could report its observed state under `status`.

The simple CRD in this guide defines `spec`; it does not yet define a
`status` section or implement a controller.

## 4. Install the CRD

First, check that your Kubernetes cluster is available:

``` bash
kubectl cluster-info
```

Apply the CRD:

``` bash
kubectl apply -f devopsbatch-crd.yaml
```

Check that Kubernetes registered it:

``` bash
kubectl get crd
```

You should find:

``` text
devopsbatches.devops.mudasirirshad.com
```

You can inspect its details:

``` bash
kubectl describe crd devopsbatches.devops.mudasirirshad.com
```

You can also ask Kubernetes for the resource names it now recognizes:

``` bash
kubectl api-resources --api-group=devops.mudasirirshad.com
```

At this point, Kubernetes knows the **resource type**. You have not
created a DevOpsBatch instance yet.

## 5. Create your first Custom Resource

Now create an actual batch.

Save this as `junoon-batch.yaml`:

``` yaml
apiVersion: devops.mudasirirshad.com/v1
kind: DevOpsBatch

metadata:
  name: junoon
  namespace: default

spec:
  name: Junoon
  duration: 6 months
  mode: Online
  platform: Kubernetes
```

Notice the difference:

-   `apiVersion` now uses your custom API group and version.
-   `kind` is `DevOpsBatch`, the kind registered in the CRD.
-   `metadata.name` is the Kubernetes object's name.
-   `spec` contains the batch configuration.

Apply it:

``` bash
kubectl apply -f junoon-batch.yaml
```

List the batches:

``` bash
kubectl get devopsbatches
```

You can also use a short name:

``` bash
kubectl get batches
```

Or:

``` bash
kubectl get tws
```

The aliases work because they were registered in the CRD.

Inspect your batch:

``` bash
kubectl describe devopsbatch junoon
```

Print its full YAML representation:

``` bash
kubectl get devopsbatch junoon -o yaml
```

## 6. Create more than one batch

You can create another Custom Resource using the same CRD.

For example, save this as `cloud-batch.yaml`:

``` yaml
apiVersion: devops.mudasirirshad.com/v1
kind: DevOpsBatch

metadata:
  name: cloud-batch
  namespace: default

spec:
  name: Cloud Engineering
  duration: 4 months
  mode: Hybrid
  platform: AWS
```

Apply it:

``` bash
kubectl apply -f cloud-batch.yaml
```

List both resources:

``` bash
kubectl get devopsbatches
```

You now have two objects of the same custom resource type. You did not
need to create a new CRD for the second batch.

## 7. Understand namespaces

Because your CRD says:

``` yaml
scope: Namespaced
```

you can create batches in different namespaces.

Create a namespace:

``` bash
kubectl create namespace training
```

Create or apply a batch in that namespace by setting:

``` yaml
metadata:
  name: junoon
  namespace: training
```

List batches in that namespace:

``` bash
kubectl get devopsbatches -n training
```

List batches across all namespaces:

``` bash
kubectl get devopsbatches --all-namespaces
```

Remember: a namespaced CRD defines a resource type that can have
instances in namespaces. It does not create those namespaces
automatically.

## 8. What happens if you do not create a CRD?

Try to understand the sequence:

1.  You ask Kubernetes for `devopsbatches`.
2.  Kubernetes checks whether it knows that resource type.
3.  If the CRD has not been registered, Kubernetes does not recognize
    it.
4.  After you apply the CRD, Kubernetes registers the type.
5.  You can then create and list DevOpsBatch objects.

The CRD extends the Kubernetes API. It does not create a Pod,
Deployment, database, or other infrastructure automatically.

## 9. CRD versus Custom Resource versus Operator

This is one of the most important concepts to understand.

### CRD: Defines what Kubernetes should understand

The CRD registers `DevOpsBatch` and describes its allowed fields.

### Custom Resource: Describes one desired object

The `junoon` Custom Resource says:

-   Batch name: Junoon
-   Duration: 6 months
-   Mode: Online
-   Platform: Kubernetes

Kubernetes can store and return this configuration.

### Controller: Makes something happen

Suppose you want Kubernetes to automatically create a Deployment
whenever someone creates a DevOpsBatch.

A controller could:

1.  Watch for DevOpsBatch objects.
2.  Detect a newly created batch.
3.  Read its `spec`.
4.  Create or update a Deployment based on that configuration.
5.  Update the resource's `status` to report progress.

An **Operator** is a common pattern for packaging a controller together
with domain-specific operational knowledge. The CRD defines the API; the
controller implements the behavior.

Without a controller, your DevOpsBatch is essentially configuration
stored in the Kubernetes API. That can still be useful, but it will not
automatically launch a training platform.

## 10. Useful kubectl commands

  --------------------------------------------------------------------------------------------------
  Command                                                        What it does
  -------------------------------------------------------------- -----------------------------------
  `kubectl get crd`                                              Lists installed CRDs

  `kubectl get devopsbatches`                                    Lists DevOpsBatch objects in the
                                                                 current namespace

  `kubectl get batches`                                          Lists them using a short name

  `kubectl get tws`                                              Lists them using another short name

  `kubectl get devopsbatches -A`                                 Lists objects across all namespaces

  `kubectl describe devopsbatch junoon`                          Shows details about one batch

  `kubectl get devopsbatch junoon -o yaml`                       Prints one batch as YAML

  `kubectl api-resources --api-group=devops.mudasirirshad.com`   Lists resources in your API group

  `kubectl delete devopsbatch junoon`                            Deletes one batch object

  `kubectl delete -f junoon-batch.yaml`                          Deletes the object described by
                                                                 that file
  --------------------------------------------------------------------------------------------------

Deleting a Custom Resource object does not delete the CRD. Deleting the
CRD removes the custom resource type from the API and also deletes its
stored custom resource objects. Be careful with this operation.

To remove only the test objects:

``` bash
kubectl delete -f junoon-batch.yaml
kubectl delete -f cloud-batch.yaml
```

To remove the CRD itself (this also removes all objects of that custom
resource type):

``` bash
kubectl delete -f devopsbatch-crd.yaml
```

## 11. Common beginner mistakes

### Mistake 1: Creating a Custom Resource before its CRD

If you apply `junoon-batch.yaml` before installing the CRD, Kubernetes
does not know the `DevOpsBatch` kind.

**Fix:** Apply the CRD first, then apply the Custom Resource.

### Mistake 2: Using the wrong API group

The Custom Resource must use the same group and version declared in the
CRD:

``` yaml
apiVersion: devops.mudasirirshad.com/v1
```

### Mistake 3: Using the wrong kind

The `kind` in the Custom Resource must match the CRD's `kind`:

``` yaml
kind: DevOpsBatch
```

### Mistake 4: Putting fields in the wrong place

This guide defines the custom fields under `spec`, so your Custom
Resource must put them under `spec` too.

### Mistake 5: Expecting a CRD to run workloads

A CRD only defines and registers the resource API. To create Deployments
or perform other automated work in response to a DevOpsBatch, you need a
controller or Operator.

## 12. Optional improvement: Validate required fields

The current schema checks that provided fields have the correct types,
but it does not require every field to be present.

You can make fields mandatory by adding `required`:

``` yaml
openAPIV3Schema:
  type: object
  properties:
    spec:
      type: object
      required:
        - name
        - duration
        - mode
        - platform
      properties:
        name:
          type: string
        duration:
          type: string
        mode:
          type: string
        platform:
          type: string
```

With this validation, Kubernetes rejects a DevOpsBatch if one of those
required fields is missing.

You can also restrict values. For example, if `mode` should only be
`Online`, `Onsite`, or `Hybrid`, you can use an enum:

``` yaml
mode:
  type: string
  enum:
    - Online
    - Onsite
    - Hybrid
```

These are schema validations. They still do not implement application
behavior.

## 13. Practice exercise

Try these tasks in your own development cluster:

1.  Apply the `devopsbatch-crd.yaml` file.
2.  Confirm that the CRD exists using `kubectl get crd`.
3.  Create the `junoon` Custom Resource.
4.  List it using `kubectl get devopsbatches`.
5.  List it using `kubectl get batches`.
6.  Create a second batch called `cloud-batch`.
7.  Create a namespace called `training`.
8.  Create a batch in the `training` namespace.
9.  List all batches across namespaces.
10. Add required-field validation to the CRD and test what happens when
    you omit `platform`.

### Questions to check your understanding

1.  Does a CRD create an actual batch, or does it define a new resource
    type?
2.  What is the difference between a CRD and a Custom Resource?
3.  Why does the Custom Resource use
    `apiVersion: devops.mudasirirshad.com/v1`?
4.  What does `scope: Namespaced` mean?
5.  Will creating a DevOpsBatch automatically create a Deployment?
6.  What component would you build if you wanted Kubernetes to take
    action whenever a DevOpsBatch is created?

## 14. Final summary

Remember these five points:

1.  Kubernetes has built-in resources such as Pods, Deployments, and
    Services.
2.  A CRD extends the Kubernetes API with a new resource type.
3.  A Custom Resource is an actual instance of that type.
4.  A schema tells Kubernetes which fields and values are valid.
5.  A controller or Operator adds automation by watching Custom
    Resources and taking action.

**The key idea:** CRDs let you teach Kubernetes about resources specific
to your own application or infrastructure. Controllers let you automate
what Kubernetes should do with those resources.
