# ocm-ramen-samples

OCM Stateful application samples, including Ramen resources.

## Initial setup

1. Clone this git repository to get started:

   ```
   git clone https://github.com/RamenDR/ocm-ramen-samples.git
   cd ocm-ramen-samples
   ```

1. Switch kubeconfig to point to the OCM Hub cluster

   ```
   kubectl config use-context hub
   ```

1. Create DRClusters and DRPolicy

   When using the ramen testing environment this is not needed, but if
   you are using your own Kubernetes clusters you need to create the
   resources.

   Modify the DRCluster and DRpolicy resources in the `ramen` directory
   to match the actual cluster names in your environment, and apply
   the kustomization:

   ```
   kubectl apply -k ramen
   ```

   This creates DRPolicy and DRCluster resources in the cluster
   namespace that can be viewed using:

   ```
   kubectl get drcluster,drpolicy
   ```

1. Setup the common OCM channel resources on the hub:

   ```
   kubectl apply -k channel
   ```

   This creates a Channel resource in the `ramen-samples` namespace and
   can be viewed using:

   ```
   kubectl get channel ramen-gitops -n ramen-samples
   ```

## Sample applications

In the workloads directory provides samples that can be deployed on
Kubernetes and OpenShift.

- deployment - busybox deployment
- kubevirt
  - vm-pvc - PVC based VM
  - vm-dv - DataVolume based VM
  - vm-dvt - DataVolumeTemplate based VM

## Managed and discovered applications

*Ramen* can protect *OCM managed applications* and *OCM discovered
applications*.

### OCM managed applications

Deployed and undeployed by *OCM*, using either *Subsription* or
*ApplciationSet* resources. *Ramen* take ownership of the application
when enabling DR, and return ownership to OCM when disabling DR. When DR
is enabled, *Ramen* control the placement of the application and backup
and restore the application PVCs.

When failing over or relocating an *OCM managed application* to another
cluster, *OCM* delete the application from the cluster and deploy it on
the other cluster.

### OCM discovered applications*

Deployed and undeployed by a user on a cluster using declarative or
imperative means. An example is deploying one of the workloads in this
repository using kubectl. Another example is creating a virtual machine
using KubeVirt console.

*Ramen* take ownership of the application when enabling DR and return
ownership of the application when disabling DR. When DR is enabled,
*Ramen* control the application placement and backup and restore the
application resources based provided *DRPlacementControl* resoruce.

When failing over or relocating a discovered application to another
cluster, *Ramen* deploy the application on the other cluster. However to
complete the operation, the user must delete the application from the
cluster since *Ramen* does not support deleting applications.

## Deploying an OCM managed application

In the example we use the busybox deployment for Kubernetes regional DR
environment using RBD storage:

    subscription/deployment-k8s-regional-rbd

This application is deployed in the `deployment-rbd` namespace on the
hub and managed clusters.

You can use other overlays to deploy on other cluster types or use
different storage class. You can also create your own overlays based on
the examples.

1. Deploy an OCM application subscription on hub:

   ```
   kubectl apply -k subscription/deployment-k8s-regional-rbd
   ```

   This creates the required Subscription, Placement, and
   ManagedClusterSetBinding resources for the deployment in the
   `deployment-rbd` namespace and can be viewed using:

   ```
   kubectl get subscription,placement -n deployment-rbd
   ```

1. Inspect subscribed resources from the channel created in the same
   namespace on the ManagedCluster selected by the Placement.

   The busybox deployment Placement `status` can be viewed on the hub
   using:

   ```
   kubectl get placement placement -n deployment-rbd
   ```

   The Busybox deployment subscribed resources, like the pod and the PVC
   can be viewed on the ManagedCluster using (example ManagedCluster
   `dr1`):

   ```
   kubectl get pod,pvc -n deployment-rbd --context dr1
   ```

## Undeploying an OCM managed application

To undeploy an application delete the subscription overlay used to
deploy the application:

```
kubectl delete -k subscription/deployment-k8s-regional-rbd
```

## Enable DR for a deployed OCM managed application

1. Change the Placement to be reconciled by Ramen

   ```
   kubectl annotate placement placement -n deployment-rbd \
      cluster.open-cluster-management.io/experimental-scheduling-disable=true
   ```

1. Deploy a DRPlacementControl resource for the OCM application on the
   hub, for example:

   ```
   kubectl apply -k dr/managed/deployment-k8s-regional-rbd
   ```

   This creates a DRPlacementControl resource for the busybox deployment
   in the `deployment-rbd` namespace and can be viewed using:

   ```
   kubectl get drpc -n deployment-rbd
   ```

   At this point the placement of the application is managed by *Ramen*.

## Disable DR for a DR enabled OCM managed application

1. Ensure the placement is pointing to the cluster where the workload is
   currently placed to avoid data loss if OCM moves the application to
   another cluster.

   The sample `placement` does not require any change, but if you are
   using an application created by OpenShift Console, you may need to
   change the cluster name in the placement.

1. Delete the drpc resource for the OCM application on the hub:

   ```
   kubectl delete -k dr/managed/deployment-k8s-regional-rbd
   ```

   This deletes the DRPlacementControl resource for the busybox
   deployment, disabling replication and removing replicated data.

1. Change the Placement to be reconciled by OCM

   ```
   kubectl annotate placement placement -n deployment-rbd \
       cluster.open-cluster-management.io/experimental-scheduling-disable-
   ```

   At this point the application is managed again by *OCM*.

## Deploy OCM discovered application

The sample application is configured to run on cluster `dr1`. To deploy
it on cluster `dr1` and make it possible to fail over or relocate to
cluster `dr2` we need to create the namespace on both clusters:

```
kubectl create ns deployment-rbd --context dr1
kubectl create ns deployment-rbd --context dr2
```

To deploy the application apply the deployment-rbd workload to the
`deployment-rbd` namespace on cluster `dr1`:

```
kubectl apply -k workloads/deployment/k8s-regional-rbd -n deployment-rbd --context dr1
```

To view the deployed application use:

```
kubectl get deploy,pod,pvc -n deployment-rbd --context dr1
```

Example output:

```
NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/busybox   1/1     1            1           24s

NAME                           READY   STATUS    RESTARTS   AGE
pod/busybox-6bbf88b9f8-fz2kn   1/1     Running   0          24s

NAME                                STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS      VOLUMEATTRIBUTESCLASS   AGE
persistentvolumeclaim/busybox-pvc   Bound    pvc-c45a3892-167b-4dbc-a250-09c5f288c766   1Gi        RWO            rook-ceph-block   <unset>                 24s
```

## Enabling DR for OCM discovered application

Unlike OCM managed applications, the DR resources for all applications
are in the `ramen-ops` namespace.

To prepare the `ramen-ops` namespaces apply the managed clusterset
binding resource. This should be done once before enabling DR for
discovered applications.

```
kubectl apply -f dr/discovered/ramen-ops/binding.yaml --context hub
```

Example output:

```
managedclustersetbinding.cluster.open-cluster-management.io/default created
```

To enable DR for the application, apply the DR resources to the hub
cluster:

```
kubectl apply -k dr/discovered/deployment-rbd --context hub
```

Example output:

```
placement.cluster.open-cluster-management.io/deployment-rbd-placement created
placementdecision.cluster.open-cluster-management.io/deployment-rbd-placement-decision created
drplacementcontrol.ramendr.openshift.io/deployment-rbd-drpc created
```

To set the application placement, patch the placement decision resource
in the `ramern-ops` namespace on the hub:

```
kubectl patch placementdecision deployment-rbd-placement-decision \
    --subresource status \
    --patch '{"status": {"decisions": [{"clusterName": "dr1", "reason": "dr1"}]}}' \
    --type merge \
    --namespace ramen-ops \
    --context hub
```

Example output:

```
placementdecision.cluster.open-cluster-management.io/deployment-rbd-placement-decision patched (no change)
```

At this point *Ramen* take over and start protecting the application.

*Ramen* creates a `VolumeReplicationGroup` resource in the `ramen-ops`
namespace in cluster `dr1`:

```
kubectl get vrg -l app=deployment-rbd -n ramen-ops --context dr1
```

Example output:

```
$ kubectl get vrg deployment-rbd-drpc -n ramen-ops --context dr1
NAME                  DESIREDSTATE   CURRENTSTATE
deployment-rbd-drpc   primary        Primary
```

*Ramen* also creates a `VolumeReplication` resource, setting up
replication for the application PVC from the primary cluster to the
secondary cluster:

```
kubectl get vr busybox-pvc -n deployment-rbd --context dr1
```

Example output:

```
NAME          AGE   VOLUMEREPLICATIONCLASS   PVCNAME       DESIREDSTATE   CURRENTSTATE
busybox-pvc   10m   vrc-sample               busybox-pvc   primary        Primary
```

## Failing over an OCM discovered application

In case of disaster you can force the application to run on the other
cluster.  The application will start on the other cluster using the data
from the last replication. Data since the last replication is lost.

In the ramen testing environment we can simulate a disaster by pausing
the minikube VM running cluster `dr1`:

```
virsh -c qemu:///system suspend dr1
```

Example output:

```
Domain 'dr1' suspended
```

At this point the application is not accessible. To recover from the
disaster, we can fail over the application the secondary cluster.

To start a `Failover` action, patch the application `DRPlacementControl`
resource in the `ramen-ops` namespace on the hub cluster. We need to set
the `action` and `failoverCluster`:

```
kubectl patch drpc deployment-rbd-drpc \
    --patch '{"spec": {"action": "Failover", "failoverCluster": "dr2"}}' \
    --type merge \
    --namespace ramen-ops \
    --context hub
```

Example output:

```
drplacementcontrol.ramendr.openshift.io/deployment-rbd-drpc patched
```

The application will start on the failover cluster ("dr2"). Nothing will
change on the primary cluster ("dr1") since it is still paused.

To watch the application status while failing over, run:

```
kubectl get drpc deployment-rbd-drpc -n ramen-ops --context hub -o wide -w
```

Example output:

```
NAME                  AGE   PREFERREDCLUSTER   FAILOVERCLUSTER   DESIREDSTATE   CURRENTSTATE   PROGRESSION        START TIME             DURATION   PEER READY
deployment-rbd-drpc   17m   dr1                dr2               Failover       FailedOver     WaitForReadiness   2024-06-04T18:10:44Z              False
deployment-rbd-drpc   18m   dr1                dr2               Failover       FailedOver     WaitForReadiness   2024-06-04T18:10:44Z              False
deployment-rbd-drpc   18m   dr1                dr2               Failover       FailedOver     Cleaning Up        2024-06-04T18:10:44Z              False
deployment-rbd-drpc   18m   dr1                dr2               Failover       FailedOver     WaitOnUserToCleanUp   2024-06-04T18:10:44Z              False
```

*Ramen* will proceed until the point where the application should be
deleted from the primary cluster ("dr1"). Note the progression
`WaitOnUserToCleanup`.

The application is running now on cluster `dr2`:

```
kubectl get deploy,pod,pvc -n deployment-rbd --context dr2
```

Example output:

```
NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/busybox   1/1     1            1           3m58s

NAME                           READY   STATUS    RESTARTS   AGE
pod/busybox-6bbf88b9f8-fz2kn   1/1     Running   0          3m58s

NAME                                STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS      VOLUMEATTRIBUTESCLASS   AGE
persistentvolumeclaim/busybox-pvc   Bound    pvc-c45a3892-167b-4dbc-a250-09c5f288c766   1Gi        RWO            rook-ceph-block   <unset>                 4m11s
```

To complete the failover, we need to recover the primary cluster, so we
can start replication from the secondary cluster to the primary cluster.

In the ramen testing environment, we can resume the minikube VM running
cluster `dr1`:

```
virsh -c qemu:///system resume dr1
```

Example output:

```
Domain 'dr1' resumed
```

When the cluster becomes accessible again, you need to delete the
application from the primary cluster since *Ramen* does not support
deleting applications:

```
kubectl delete -k workloads/deployment/k8s-regional-rbd -n deployment-rbd --context dr1
```

Example output:

```
persistentvolumeclaim "busybox-pvc" deleted
deployment.apps "busybox" deleted
```

To wait until the application data is replicated again to the other
cluster run:

```
kubectl wait drpc deployment-rbd-drpc \
    --for condition=Protected \
    --namespace ramen-ops \
    --timeout 5m \
    --context hub
```

Example output:

```
drplacementcontrol.ramendr.openshift.io/deployment-rbd-drpc condition met
```

To check the application DR status run:

```
kubectl get drpc deployment-rbd-drpc -n ramen-ops --context hub -o wide
```

Example output:

```
NAME                  AGE   PREFERREDCLUSTER   FAILOVERCLUSTER   DESIREDSTATE   CURRENTSTATE   PROGRESSION   START TIME             DURATION          PEER READY
deployment-rbd-drpc   28m   dr1                dr2               Failover       FailedOver     Completed     2024-06-04T18:10:44Z   11m24.41686883s   True
```

The failover has completed, and the application data is replicated again
to the primary cluster.

## Relocate an OCM discovered application

To move the application back to the primary cluster after a disaster you
can use the `Relocate` action. You will delete the application on the
secondary cluster, and *Ramen* will start it on the primary cluster. No
data is lost during this operation.

To start the relocate operation, patch the application
`DRPlacementControl` resource in the `ramen-ops` namespace on the hub.
We need to set `action` and if needed, `preferredCluster`:

```
kubectl patch drpc deployment-rbd-drpc \
    --patch '{"spec": {"action": "Relocate", "preferredCluster": "dr1"}}' \
    --type merge \
    --namespace ramen-ops \
    --context hub
```

Example output:

```
drplacementcontrol.ramendr.openshift.io/deployment-rbd-drpc patched
```

*Ramen* will prepare for relocation, and proceed until the point the
application should be deleted from the cluster. To watch the progress
run:

```
kubectl get drpc deployment-rbd-drpc -n ramen-ops --context hub -o wide -w
```

Example output:

```
NAME                  AGE   PREFERREDCLUSTER   FAILOVERCLUSTER   DESIREDSTATE   CURRENTSTATE   PROGRESSION          START TIME             DURATION   PEER READY
deployment-rbd-drpc   91m   dr1                dr2               Relocate       Initiating     PreparingFinalSync   2024-06-04T19:25:52Z              True
deployment-rbd-drpc   92m   dr1                dr2               Relocate       Relocating     RunningFinalSync     2024-06-04T19:25:52Z              True
deployment-rbd-drpc   92m   dr1                dr2               Relocate       Relocating     WaitOnUserToCleanUp   2024-06-04T19:25:52Z              False
```

When ramen shows the progression `WaitOnUserToCleanUp` you need to
delete the application from the secondary cluster:

```
kubectl delete -k workloads/deployment/k8s-regional-rbd -n deployment-rbd --context dr2
```

Example output:

```
persistentvolumeclaim "busybox-pvc" deleted
deployment.apps "busybox" deleted
```

At this pint *Ramen* will proceed with starting the application on the
primary cluster, and setting up replication to the secondary cluster.

To wait until the application is relocated to the primary cluster, run:

```
kubectl wait drpc deployment-rbd-drpc \
    --for jsonpath='{.status.phase}=Relocated' \
    --namespace ramen-ops \
    --timeout 5m \
    --context hub
```

Example output:

```
drplacementcontrol.ramendr.openshift.io/deployment-rbd-drpc condition met
```

To wait until the application is replicating data again to the secondary
cluster, wait for the `Protected` condition:

```
kubectl wait drpc deployment-rbd-drpc \
    --for condition=Protected \
    --namespace ramen-ops \
    --timeout 5m \
    --context hub
```

Example output:

```
drplacementcontrol.ramendr.openshift.io/deployment-rbd-drpc condition met
```

To check the application DR status run:

```
kubectl get drpc deployment-rbd-drpc -n ramen-ops --context hub -o wide
```

Example output:

```
NAME                  AGE    PREFERREDCLUSTER   FAILOVERCLUSTER   DESIREDSTATE   CURRENTSTATE   PROGRESSION   START TIME             DURATION          PEER READY
deployment-rbd-drpc   103m   dr1                dr2               Relocate       Relocated      Completed     2024-06-04T19:25:52Z   3m46.040128192s   True
```

The relocate has completed, and the application data is replicated again
to the secondary cluster.

## Disable DR for a DR enabled OCM discovered application

Since OCM is not managing the application, we can simply delete the DR
resources:

```
kubectl delete -k dr/discovered/deployment-rbd --context hub
```

Example output:

```
placement.cluster.open-cluster-management.io "deployment-rbd-placement" deleted
placementdecision.cluster.open-cluster-management.io "deployment-rbd-placement-decision" deleted
drplacementcontrol.ramendr.openshift.io "deployment-rbd-drpc" deleted
```

This deletes the DR resources, disable replication and delete the
replicated data on the secondary cluster.

The application is still running on the primary cluster:

```
$ kubectl get deploy,pod,pvc -n deployment-rbd --context dr1
```

Example output:

```
NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/busybox   1/1     1            1           16m

NAME                           READY   STATUS    RESTARTS   AGE
pod/busybox-6bbf88b9f8-fz2kn   1/1     Running   0          16m

NAME                                STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS      VOLUMEATTRIBUTESCLASS   AGE
persistentvolumeclaim/busybox-pvc   Bound    pvc-c45a3892-167b-4dbc-a250-09c5f288c766   1Gi        RWO            rook-ceph-block   <unset>                 16m
```

## Undeploy an OCM discovered application

Delete the application workload from the cluster:

```
kubectl delete -k workloads/deployment/k8s-regional-rbd -n deployment-rbd --context dr1
```

Example output:

```
persistentvolumeclaim "busybox-pvc" deleted
deployment.apps "busybox" deleted
```
