# Resouce utilization and management demos

The commands for the demos assume you are in the `demos` folder.  If you have not cloned the repository and changed directories, please do that beforehand.

## Resource requests and quotas

1. Create a project for this demo
   
   ```bash
   # create a project
   oc adm new-project starwars
   
   # switch to the project
   oc project starwars
   ```
   
1. Create the limit range and quota
   
   Create the objects: `oc create -f starwars/limit-and-quota.yaml`. 
   
   View the created limit range using the commands `oc get limits` and `oc describe limits default`.
   
   View the created quota using the commands `oc get quota default` and `oc describe quota default`.
   
1. Create a pod with no resource request or limit
   
   Examine the file `starwars/millenium-falcon.yaml`.  Create the pod: `oc create -f starwars/millenium-falcon.yaml`.  
   
   Review the created pod and containers using the command `oc describe pod millenium-falcon`.
   
   The expected result is that the two containers, `han` and `chewy`, each have *requests* and *limits* which match the default values in the limit range.  The QoS class is `Burstable` because of the defined resource constraints.
   
1. Create a pod which only defines a limit
   
   Examine the file `starwars/x-wing.yaml`.  Create the pod: `oc create -f starwars/x-wing.yaml`.
   
   Review the created pod and container using the command `oc describe pod x-wing`.
   
   The expected result is that the pod has a *request* and *limit* which matches the pod definition.  A QoS class of `Graranteed` has been assigned because the request and limit are the same.
   
1. Create a pod which exceeds the limit
   
   Examine the file `starwars/deathstar.yaml`.  Create the pod: `oc create -f starwars/deathstar.yaml`.
   
   The expected result is that OpenShift refuses to create the pod because it exceeds the maximum defined in the limit range.
   
1. Create pods which will exceed the quota
   
   Review the current quota usage: `oc describe quota default`.

   Examine the file `starwars/stardestroyers.yaml`.  Create the pods: `oc create -f starwars/stardestroyers.yaml`.
   
   View the created pods using the command `oc get pod -l app=galacticEmpire`.

   The expected result is that the first three pods are created, but the last is not.  The last pod will exceed the quota for the project.
   
1. Destroy the project
   
   Use the command `oc delete project starwars`.

## Pod Scheduling 

This demo will highlight pod affinity and anti-affinity, node selectors and affinity, and taints/tolerations.  The cluster being used must have at least three nodes, substitute your node names for `<node 1>`, `<node 2>`, and `<node 3>`.

To view worker nodes in your cluster, use the command `oc get node -l node-role.kubernetes.io/worker=`.

```console
[you@bastion ~]$ oc get node -l node-role.kubernetes.io/worker=
NAME                                         STATUS   ROLES    AGE   VERSION
ip-10-0-130-35.us-east-2.compute.internal    Ready    worker   88m   v1.12.4+4dd65df23d
ip-10-0-155-172.us-east-2.compute.internal   Ready    worker   88m   v1.12.4+4dd65df23d
ip-10-0-165-66.us-east-2.compute.internal    Ready    worker   88m   v1.12.4+4dd65df23d
```

1. Add node labels and taints, create the project
   
   ```bash
   # add node labels
   oc label node <node 1> glutenFree=false
   oc label node <node 2> glutenFree=true
   oc label node <node 3> glutenFree=false
   ```
   
   We also need to create a project for this demo:

   ```bash
   # create the project
   oc adm new-project sandwiches
   
   # switch to the project
   oc project sandwiches
   ```

1. Review the `turkey` pod, examine the node selector and tolerations.  The tolerations are not important to scheduling now, but will be later.
   
   ```bash
   # review the definition
   cat sandwiches/turkey.yaml
   
   # create it
   oc create -f sandwiches/turkey.yaml
   
   # view the scheduled node
   oc get pod -o wide
   ```
   
   The expected result is that the pod is scheduled to node 2 as a result of the node selector.
   
1. Review the `pbandj` pod.
   
   ```bash
   # review the definition
   cat sandwiches/pbandj.yaml

   # create the pod
   oc create -f sandwiches/pbandj.yaml

   # view the scheduled node
   oc get pod -o wide
   ```
   
   This pod has an anit-affinity rule which prevents it from being executed on the same node as any pod with a `meatType` label defined, no other scheduling rules are defined.
   
   The pod is expected to scheule on `<node 1>` or `<node 3>`.
   
1. Review the `grilledcheese` pod.
   
   ```bash
   # review the definition
   cat sandwiches/grilledcheese.yaml
   
   # create the pod
   oc create -f sandwiches/grilledcheese.yaml
   
   # show where it was scheduled to
   oc get pod -o wide
   ```
   
   This pod has a *preferred* affinity rule for a pod with a `cheeseType` label and a *required* anti-affinity rule for a node with pods which have *peanut* or *almond* `nutType` labels. The pod is expected to scheule on `<node 2>` because of the affinity rule.
   
1. Add a taint to `<node 2>` of `breadType=wheat:NoExecute`.
   
   ```bash
   # taint the node
   oc adm taint node <node 2> breadType=wheat:NoExecute

   # watch the pods
   oc get pod -o wide -w
   ```

   The result will be that the `grilledcheese` pod will be terminated.  It does not have a toleration for that taint.  Recreate the `grilledcheese` pod.

   ```bash
   # create the pod
   oc create -f sandwiches/grilledcheese.yaml

   # view the scheduled node
   oc get pod -o wide
   ```
   
   The scheduler will not schedule the pod to the same node as `pbandj` due to the anti-affinity, and the tainted node will repel the pod.  The result is the only remaining node in the cluster.