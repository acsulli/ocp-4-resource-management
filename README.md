# OpenShift 4 Resource Management

This module will review and explore resource management concepts in OpenShift 4.

* [Resource Overview](#resource-overview)
  * [Resource requests and limits](#limits-and-requests)
  * [Limit ranges](#limit-range)
  * [Quotas](#quotas)
  * [QoS](#quality-of-service-qos)
* [Scheduling](#scheduling)
  * [Pod affinity and anti-affinity](#pod-affinity-and-anti-affinity)
  * [Node selectors](#node-selectors)
  * [Node affinity](#node-affinity)
  * [Taints and tolerations](#taints-and-tolerations)

Demos for these concepts are available from the [demos folder](demos/README.md).

## Resources overview

The basic unit of execution for OpenShift (and Kubernetes) is the pod.  A pod, at its simplest, is one or more containers.  Each container in the pod may have resource requests and limits assigned to it as a part of the pod definition or inherit a default from the project-scoped limit range.  The total resouce consumption, for all pods in the project, is controlled by quotas.

* CPU
  
  CPU requests are specified in millicores, or 1/1000th of a CPU core, but can be shortened to less granular decimal numbers if desired.  For example, one half of one CPU core would be `500m` or `.5`, 1.5 cores would be `1500m` or `1.5`.
  
* RAM
  
  RAM requests are in bytes, by default, but can use standard suffixes to indicate more usable units.  For example, 1 GB = `1G`, 512MB = `512M`, etc.
  
  Note that the power of 10 numbers (MB, GB, TB) and power of 2 numbers (MiB, GiB, TiB) are honored when making resource requests.  One (1) GiB == approx 1.07 GB == 1024 MiB == approx 1074 MB.

Each of these resources may have a *request*, which indicates the minimum guaranteed amount of CPU/RAM which must be allocated (and is used for scheduling the pod), and a *limit*, which indicates the maximum amount of CPU/RAM the pod is allowed to utilize.

### Limits and Requests

The request and limit are provided in the pod definition:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: minecraft-pod
spec:
  containers:
  - name: minecraft
    image: [...]
    resources:
      limits:
        cpu: .5
        memory: "512Mi"
      requests:
        cpu: 1
        memory: "1Gi"
[...]
```

If no nodes have enough resources to schedule the pod, OpenShift will fail to schedule and continue to retry until the pod is deleted or resources become avaialble.  Exceeding the limits can have ramifications, such as eviction or termination, particularly if the node is experiencing CPU or RAM pressure.

### Limit Range

Limit ranges provide a range of valid values, normally both a lower and upper boundry, for resources such as CPU and RAM.  They are defined on a per-project basis.  Default values may be provided by the administrator if the pod creator does not specify values.

An example limit range object looks like this:

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limit-range
spec:
  limits:
  - max:
      memory: 2Gi
      cpu: 2
    min:
      memory: 16Mi
      cpu: 50m
    default:
      memory: 1Gi
      cpu: 1
    defaultRequest:
      memory: 256Mi
      cpu: 250m
    type: Container
```

In this example, we have specified the following for each container in the namespace:

* No more than 2GiB of RAM and 2 CPUs may be requested 
* No less than 16MiB of RAM and .05 CPU may be requested
* If no RAM request or limit is specified, the pod will recieve default values of a 256MiB request with a 1GiB limit
* If no CPU request or limit is specified, the pod will receive default values of a .25 CPU request with a 1 CPU limit

The last line of the limit range definition reads `type: Container`, which tells OpenShift to apply these values to each container.  However, using the value `type: Pod` indicates that the values should apply to the pod, meaning the sum resource requests and limits of the constituent pods cannot exceed the values.  OpenShift allows both container and pod limit ranges to co-exist and will adhere to both when defined.

Lastly, there are different behaviors if only one of the request or limit is specified:

* A *limit* is defined, but no *request* is defined - The *request* will be set to the same as the *limit* for the running pod.
* A *request* is defined, but no *limit* is defined - The default *limit* will be assigned to the pod.

Viewing limit ranges is done using the command `oc describe limits`.

Setting a low default may result in poor performance when the nodes are highly overcommitted.  OpenShift will not terminate or evict low priority pods to free resources until contention occurrs, so it's important to set sane defaults and rely on pod owners to request values which are appropriate for their application.

### Quotas

As discussed above, cluster resources, meaning CPU and RAM, are allocated using *limits* and *requests* defined as a part of the pod spec.  The values for those limits and requests are controlled by a *limit range*.  However, this does not provide controls to the total amount of resources which may be consumed by a project.

To provide controls on the total amount of CPU and RAM which may be consumed, *quotas* are used.  As the name implies, this defines the enforced limit on the quantity of CPU and RAM which is available to the pods in a project.

Quotas and limit ranges work together to achieve granualar allocation and control of the finite resources in the cluster.  The quota determines the maximum quanity which can be consumed, e.g. "the project may consume no more than 5GB of RAM", whereas the limit range determines the (min and max) increments which that quantity is provisioned in.  These two objects work together to ensure that no user or project consumes more resources than allocated, and no single (or small number of, depending on configuration) request may consume more than it's reasonable amount of resources.

Below is an example quota for CPU and RAM limits:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: resource-quota
spec:
  hard:
    requests.cpu: "32"
    requests.memory: 64Gi
    limits.cpu: "64"
    limits.memory: 128Gi
```

Like a limit range, both requests (the minimum amount of resources desired) and limits (the maximum amount of resources allowed) are defined in the quota.

In addition to controlling physical resources of the cluster, quotas can also limit the number of API objects which the project may create.  An API object refers to Kubernetes/OpenShift objects such as *pods*, *secrets*, *configmaps*, *services*, and more.  Specifying limits for objects using a quota is the same as for resources:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: object-quota
spec:
  hard:
    pods: "30"
    secrets: "10"
    services: "3"
```

Creating a request which will exceed a quota will result in an error message and the pod, or other object, not being created.  Viewing the status of a quota, i.e. the limit and the amount consumed, is done using the command `oc describe quota`.

### Quality of Service (QoS)

Depending on the request and/or limit values which are specified with the pod definition (or inherited), different levels of QoS for resources will be assigned to the pod.  It's important to point out that this is not QoS in the same sense as a network, but rather how the scheduler will allocate and guarantee the resources for the pod.

The three classes of QoS, in decending order of precedence, are:

* `Guaranteed` - This class is assigned when the request and limit are the same, e.g. "I have requested 1GB of RAM and my limit is 1GB of RAM".  The pod will not be terminated unless it exceeds the limit.
* `Burstable` - A pod with a request that is less than the limit, e.g. "I am requesting 512MB of RAM, but my limit is 1GB", is assigned this class.  These pods may be terminated if the node comes under resource pressure and no lower priority pods exist.
* `BestEffort` - Pods which have no request or limit specified are assigned to this class.  They are treated as the lowest priority and will be terminated if the node comes under resource pressure, however they can consume any amount of free resources in the node until that time.

QoS is important because it expresses how critical the resources for the pod are.  A pod with a `BestEffort` class may be terminated to make room for higher priority pods and not rescheduled until more resources become available, which could be never.  A pod with a class of `Guaranteed` will not be terminated to make resources avaialble for other pods.

QoS classes are automatically determined based on the limit and request defined (or not, as the case may be).  To view the assigned class for a pod, simply use the `oc describe pod` command.

```console
[you@bastion ~]$ oc describe pod grafana-754d4bf6bc-4x92x -n openshift-monitoring
Name:               grafana-754d4bf6bc-4x92x
Namespace:          openshift-monitoring
[...]
Containers:
  grafana:
    [...]
    Limits:
      cpu:     200m
      memory:  200Mi
    Requests:
      cpu:        100m
      memory:     100Mi
[...]
QoS Class:       Burstable
[...]
```

In the above example, the request (`100m` of CPU, `100Mi` of RAM) is less than the limit (`200m` of CPU, `200Mi` of ), resulting in a QoS class of `Burstable`.

## Scheduling

Scheduling, in this context, refers to configuring OpenShift, and/or the pods, so that they have a specific behavior:

* Pod affinity - place pods on the same node
* Pod anti-affinity - do not place pods on the same node
* Node affinity - place pods on matching nodes

Rules affecting affinity and anti-affinity may have one or both of *required* and *preferred* options.  As the name implies, a required (anti-)affinity rule must have it's conditions satisfied, whereas a preferred rule is attempted to be met on a best effort basis.

In addition to individual pods, projects may be configured with constraints as well.  This is useful, for example, when an insecure project should not be executed on the same hardware as a secure project, i.e. `project1` is only allowed to run on `node1`, `project2` is only allowed to run on `node2`, but `project3` may run on either `node1` or `node2`.

### Pod affinity and anti-affinity

Affinity is useful for pods which need to be co-located, perhaps for performance or security reasons.  Anti-affinity has the opposite affect, causing the pods to repel eachother.  

To illustrate pod affinity, we first must create a pod.  The specific image and other information about the containers is not important for this example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: peanutButter
  labels:
    condimentType: "nutbutter"
    nutType: "peanut"
spec:
[...]
```

To have pods "attracted", via affinity, to this pod we specify an affinity rule for the additional pod(s):

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: strawberryJam
  labels:
    condimentType: "jam"
    fruit: "strawberry"
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: nutType
            operator: In
            values:
            - "peanut"
        topologyKey: kubernetes.io/hostname
  containers:
  [...]
```

There are several important concepts here.

* First is the `podAffinity` line in the `spec.affinity` section.  This indicates the type of rule, whether to attract or repel.  A value of `podAntiAffinity` would result in pods which repel eachother.
* Second is the line `requiredDuringSchedulingIgnoredDuringExecution`.  This indicates that the rule is mandatory.  If a value of `preferredDuringSchedulingIgnoredDuringExecution` is used, then the rule is a best effort to follow.
* Third is the operator for the expression: `operator: In`.  This indications how to test for the condition.  Valid values include `In`, `NotIn`, `Exists`, or `DoesNotExist`
* Last, the line for `topologyKey`.  This indicates how "closely" (or far apart) the pods will be scheduled.  There are multiple values which can be provided, the most common are:
  
  * `kubernetes.io/hostname` - affinity will be on the same host, anti-affinity will result in different hosts
  * `failure-domain.beta.kubernetes.io/zone` - affinity will be in the same availability zone, anti-affinity will result in different AZs
  * `failure-domain.beta.kubernetes.io/region` - affinity will be in the same availablility region, anti-affinity will result in different regions
  
  Note that these lables are attached to the nodes automatically, so you should ensure which ones are appropriate for your cluster before using them.

These rules can be combined and there may be more than one:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: grapeJelly
  labels:
    condimentType: "jelly"
    fruit: "grape"
spec:
  affinity:
    podAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 10
        labelSelector:
          matchExpressions:
          - key: condimentType
            operator: In
            values:
            - "nutbutter"
          - key: nutType
            operator: In
            values:
            - "peanut"
            - "almond"
        topologyKey: kubernetes.io/hostname
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: nutType
            operator: In
            values:
            - "pistachio"
        topologyKey: kubernetes.io/hostname
  containers:
  [...]
```

This will result in a *grape jelly* pod which favors running on nodes with *peanut butter* and *almond butter* pods, but will not run on pods with *pistachio butter* pods.

Notice that the *preferred* rule also has a `weight` associated with it.  After required scheduling rules have been applied, the remaining nodes have each of the preferred rules evaluated.  If the node meets the needs of the preference, then the weight value is added to it's score.  At the end, the node with the highest total weight is the one which the pod is scheduled to.  Valid values for the weight are `1-100`, so rules which are more important can have higher weights associated, causing them to have greater influence on the scheduling.

### Node selectors

A pod can request that it be scheduled against nodes which have, or not as the case may be, a specific label.  Node labels can represent any arbitrary value, however the `openshift-installer` process assigns the following node labels by default:

* `beta.kubernetes.io/arch=amd64` - this label determines the OS architecture
* `beta.kubernetes.io/os=linux` - this label provides the host OS type
* `beta.kubernetes.io/instance-type=<type>` - this label represents the size of the instance, e.g. `m4-xlarge`
* `failure-domain.beta.kubernetes.io/region=<region>` - where `<region>` equals the AWS region
* `failure-domain.beta.kubernetes.io/zone=<zone>` - where `<zone>` represents the AWS availability zone in the region
* `kubernetes.io/hostname=<hostname>` - where `<hostname>` is the node hostname
* `node-role.kubernetes.io/<role>=` - where `<role>` is either `master` or `worker`, depending on what role the node is filling

To view the labels which have been applied to your nodes, describe them (`oc describe node`) or use the command `oc get node --show-labels`.

Node selectors may be applied to pods, projects, or the entire cluster.  To specify a node selector for a pod, provide the desired label(s) in the `spec.nodeSelector` section:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: turkeySandwich
  labels:
    meatType: "turkey"
    cheeseType: "american"
    mayo: "yes"
    mustard: "yellow"
    ketchup: "no"
spec:
  nodeSelector:
    breadType: "wheat"
```

Creating the above pod will result in it running on a node which has the label `breadType=wheat`.

Node selectors and affinity work in conjunction.  If a node selector and *required* affinity rule are both specified, but no nodes have labels which match both requirements, then the pod will not be scheduled.

### Node affinity

In addition to pods attracting (affinity) or repeling (anti-affinity) eachother, pods can define node affinity rules, were the pod(s) will run on nodes matching the rules.  As with pod affinity, these rules can be defined as required or preferred.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: reuben
spec:
  affinity:
    nodeAffinity: 
      preferredDuringSchedulingIgnoredDuringExecution: 
      - weight: 10
        preference:
          matchExpressions:
          - key: breadType 
            operator: In 
            values:
            - rye
            - pumpernickel
      - weight: 40
        preference:
          matchExpressions:
          - key: cheeseType
            operator: In 
            values:
            - swiss
            - provolone
      requiredDuringSchedulingIgnoredDuringExecution: 
        nodeSelectorTerms:
        - matchExpressions:
          - key: sauerkraut 
            operator: Exists
```

Node affinity may also be defined on a per-project level by the administrator.  This allows separation between different projects, or prevents access to specialized nodes without being specifically granted.

```bash
# create a new project
oc adm new-project --node-selector=breadStatus=sliced
```

### Taints and tolerations

The other tools described for controlling the scheduling of pods are applied by the underlying Kubernetes scheduler and the logic it uses when deciding where a pod should be executed.  Taints and tolerations allow the node a measure of control with this logic.  

A taint is a property and effect attached to the node, e.g. a property of `can-toast-bread=yes` and an effect of `NoSchedule` means that pods without a matching toleration, it will not be executed.

Creating a taint is done by the administrator:

```bash
# add a taint to a node
oc adm taint node <node name> toasted-bread=yes:NoSchedule
```

There are three possible effects which can be assigned to the taint:

* NoSchedule - New pods without a toleration are not scheduled to this node.  Existing pods are unaffected.
* PreferNoSchedule - The scheduler **attempts* not to schedule pods without a matching toleration, however some conditions may lead to non-toleration pods on the host.  Existing pods are unaffected.
* NoExecute - New pods without a toleration cannot be scheduled to the node, existing pods without a matching toleration are removed.

Tolerations are defined on pods and must match both the taint and the effect:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: club-sandwich
spec:
  tolerations:
  - key: "can-toast-bread"
    operator: "Equal"
    value: "yes"
    effect: "NoSchedule"
```