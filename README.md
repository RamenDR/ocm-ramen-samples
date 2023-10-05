# ocm-ramen-samples

OCM Stateful application samples, including Ramen resources.

## Initial setup

1. Clone this git repository to get started:
    - `https://github.com/RamenDR/ocm-ramen-samples.git`
  `cd ocm-ramen-samples`
1. Switch kubeconfig to point to the OCM Hub cluster
1. If ramen is not configured, configure it using:
    - `kubectl apply -k ramen/`
    - The above creates DRPolicy and DRCluster resources in the
      cluster namespace that can be viewed using:
        - `kubectl get drcluster`
        - `kubectl get drpolicy`
    - Modify the DRCluster and DRPolicy resources to match the actual
      cluster names.
1. Setup the common OCM channel resources on the hub:
    - `kubectl apply -k channel/`
    - The above creates a Channel resource in the `ramen-samples`
      namespace and can be viewed using:
        - `kubectl get channel -n ramen-samples ramen-gitops`

## Sample application deployment

1. Deploy an OCM application and its related resources on the hub, for
  example:
    - `kubectl apply -k subscription/`
    - The above creates the required Subscription, Placement, and
    ManagedClusterSetBinding resources for the busybox application in
    the `busybox-sample` namespace and can be viewed using:
        - `kubectl get placement -n busybox-sample`
        - `kubectl get -n busybox-sample subscriptions`
1. Inspect subscribed resources from the channel created in the same namespace
  on the ManagedCluster selected by the Placement, for example:
    - The busybox sample Placement `status` can be viewed on the hub
    using:
        - `kubectl get placement -n busybox-sample busybox-placement`
    - Busybox subscribed resources, like the pod and the PVC can be viewed on
    the ManagedCluster using (example ManagedCluster `cluster1`):
        - `kubectl --context=cluster1 get pods busybox-sample`
        - `kubectl --context=cluster1 get pvc -n busybox-sample`

## Undeploying the sample application

To undeploy the application delete the subscription kustomization:
    - `kubectl delete -k subscription/`

## Enable DR for the OCM application

1. Change the Placement to be reconciled by Ramen

    ```
    kubectl annotate placement busybox-placement -n busybox-sample \
       cluster.open-cluster-management.io/experimental-scheduling-disable=true
    ```

1. Deploy a DRPlacementControl resource for the OCM application on the
   hub, for example:
    - `kubectl apply -k dr/`
    - The above creates a DRPlacementControl resource for the busybox
    application in the `busybox-sample` namespace and can be viewed
    using:
        - `kubectl get drpc -n busybox-sample`
    - At this point the application is managed by *Ramen*.

## Disable DR for the OCM application

1. Ensure Placement is pointing to the cluster where the workload is
   currently placed to avoid data loss if OCM moves the application to
   another cluster.
   - The sample `busybox-placement` does not require any change.
1. Delete the drpc resource for the OCM application on the hub, for example:
    - `kubectl delete -k dr/`
    - The above deletes the DRPlacementControl resource for the busybox
    application, disabling replication and removing replicated data.
1. Change the Placement to be reconciled by OCM

    ```
    kubectl annotate placement busybox-placement -n busybox-sample \
        cluster.open-cluster-management.io/experimental-scheduling-disable-
    ```

    - At this point the application is managed again by *OCM*.

## Testing failover

**Under construction**

## Testing failback

**Under construction**
