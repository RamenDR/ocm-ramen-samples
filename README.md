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

## Sample application deployment

1. Deploy an OCM application subscription on hub:

   ```
   kubectl apply -k subscription
   ```

   This creates the required Subscription, Placement, and
   ManagedClusterSetBinding resources for the busybox application in the
   `busybox-sample` namespace and can be viewed using:

   ```
   kubectl get subscription,placement -n busybox-sample
   ```

1. Inspect subscribed resources from the channel created in the same
   namespace on the ManagedCluster selected by the Placement.

   The busybox sample Placement `status` can be viewed on the hub using:

   ```
   kubectl get placement busybox-placement -n busybox-sample
   ```

   Busybox subscribed resources, like the pod and the PVC can be viewed
   on the ManagedCluster using (example ManagedCluster `cluster1`):

   ```
   kubectl get pod,pvc -n busybox-sample --context cluster1
   ```

## Undeploying the sample application

To undeploy the application delete the subscription kustomization:

```
kubectl delete -k subscription
```

## Enable DR for the OCM application

1. Change the Placement to be reconciled by Ramen

   ```
   kubectl annotate placement busybox-placement -n busybox-sample \
      cluster.open-cluster-management.io/experimental-scheduling-disable=true
   ```

1. Deploy a DRPlacementControl resource for the OCM application on the
   hub, for example:

   ```
   kubectl apply -k dr
   ```

   This creates a DRPlacementControl resource for the busybox
   application in the `busybox-sample` namespace and can be viewed
   using:

   ```
   kubectl get drpc -n busybox-sample
   ```

   At this point the placement of the application is managed by *Ramen*.

## Disable DR for the OCM application

1. Ensure Placement is pointing to the cluster where the workload is
   currently placed to avoid data loss if OCM moves the application to
   another cluster.

   The sample `busybox-placement` does not require any change, but if
   you are using an application created by OCP Console, you may need to
   change the cluster name in the placement.

1. Delete the drpc resource for the OCM application on the hub:

   ```
   kubectl delete -k dr
   ```

   This deletes the DRPlacementControl resource for the busybox
   application, disabling replication and removing replicated data.

1. Change the Placement to be reconciled by OCM

   ```
   kubectl annotate placement busybox-placement -n busybox-sample \
       cluster.open-cluster-management.io/experimental-scheduling-disable-
   ```

   At this point the application is managed again by *OCM*.
