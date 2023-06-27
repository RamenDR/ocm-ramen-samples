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
    - `kubectl apply -k subscriptions/`
    - The above creates a Channel resource in the `ramen-samples`
      namespace and can be viewed using:
        - `kubectl get channel -n ramen-samples ramen-gitops`

## Sample application deployment

1. Deploy an OCM application and its related resources on the hub, for
  example:
    - `kubectl apply -k subscriptions/busybox/`
    - The above creates the required Subscription, PlacementRule and
    DRPlacementControl resources for the busybox application in the
    `busybox-sample` namespace and can be viewed using:
        - `kubectl get placementrule -n busybox-sample`
        - `kubectl get -n busybox-sample subscriptions`
        - `kubectl get drplacementcontrol -n busybox-sample`
1. Inspect subscribed resources from the channel created in the same namespace
  on the ManagedCluster selected by the PlacementRule, for example:
    - The busybox sample PlacementRule `status` can be viewed on the hub
    using:
        - `kubectl get placementrule -n busybox-sample busybox-placement`
    - Busybox subscribed resources, like the pod and the PVC can be viewed on
    the ManagedCluster using (example ManagedCluster `cluster1`):
        - `kubectl --context=cluster1 get pods busybox-sample`
        - `kubectl --context=cluster1 get pvc -n busybox-sample`

## Testing failover

**Under construction**

## Testing failback

**Under construction**
