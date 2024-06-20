# Experiment with OCM managed application using RBD storage

In the example we use the busybox deployment for Kubernetes regional DR
environment using RBD storage:

    subscription/deployment-k8s-regional-rbd

This application is deployed in the `deployment-rbd` namespace on the
hub and managed clusters.

You can use other overlays to deploy on other cluster types or use
different storage class. You can also create your own overlays based on
the examples.

## Deploying a sample application

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

## Undeploying a sample application

To undeploy an application delete the subscription overlay used to
deploy the application:

```
kubectl delete -k subscription/deployment-k8s-regional-rbd
```

## Enable DR for a deployed application

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

## Disable DR for a DR enabled application

1. Delete the drpc resource for the OCM application on the hub:

   ```
   kubectl delete -k dr/deployment-k8s-regional-rbd
   ```

   This deletes the DRPlacementControl resource for the busybox
   deployment, disabling replication and removing replicated data.

> [!IMPORTANT]
> Do not delete the Placement annotation
> `cluster.open-cluster-management.io/experimental-scheduling-disable`
> to ensure that OCM will not change the placement of the application,
> which can result in data loss.

### Optional: enabling OCM scheduling for the application

It is not recommended to enable OCM scheduling on after disabling DR,
since OCM does not support moving workload storage between clusters.  If
the placement point to wrong cluster, OCM will delete the application
and its storage from the current cluster, and deploy the application
with new storage on the cluster selected by the placement.

Find the current placement of the application:

```
kubectl get placementdecisions -n deployment-rbd --context hub \
    -o jsonpath='{.items[0].status.decisions[0].clusterName}{"\n"}'
```

Ensure that the `Placement` predicates is pointing to the cluster where
the workload is currently placed. Here is example predicates selecting
the cluster `dr1`:

```
spec:
  clusterSets:
  - default
  numberOfClusters: 1
  predicates:
  - requiredClusterSelector:
      claimSelector: {}
      labelSelector:
        matchExpressions:
        - key: name
          operator: In
          values:
          - dr1
```

Change the Placement to be reconciled by OCM:

```
kubectl annotate placement placement -n deployment-rbd \
    cluster.open-cluster-management.io/experimental-scheduling-disable-
```

At this point the application is managed again by *OCM*.
