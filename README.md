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
   kubectl apply -k dr/deployment-k8s-regional-rbd
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
   kubectl delete -k dr/deployment-k8s-regional-rbd
   ```

   This deletes the DRPlacementControl resource for the busybox
   deployment, disabling replication and removing replicated data.

1. Change the Placement to be reconciled by OCM

   ```
   kubectl annotate placement placement -n deployment-rbd \
       cluster.open-cluster-management.io/experimental-scheduling-disable-
   ```

   At this point the application is managed again by *OCM*.
