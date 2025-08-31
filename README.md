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

1. Ensure ArgoCD has permissions to read(get, list, watch) OCM resources(placement and placement decisions) -- required for ApplicationSet integration with OCM Placement.
Hint: If this permission is not there, you can deploy a ClusterRole and Rolebinding with these RBACs.

## Sample applications

In the workloads directory provides samples that can be deployed on
Kubernetes and OpenShift.

- deployment - busybox deployment
- kubevirt
  - vm-pvc - PVC based VM
  - vm-dv - DataVolume based VM
  - vm-dvt - DataVolumeTemplate based VM

## Deploying a sample application

In the example we use the busybox deployment for Kubernetes regional DR
environment using RBD storage:

    applicationset/deployment-k8s-regional-rbd

This application is deployed in the `deployment-rbd` namespace on the
hub and managed clusters.

You can use other overlays to deploy on other cluster types or use
different storage class. You can also create your own overlays based on
the examples.

1. Deploy an OCM application using ApplicationSet on hub:

   ```
   kubectl apply -k applicationset/deployment-k8s-regional-rbd
   ```

   This creates the required ApplicationSet, Placement, ConfigMap, and
   ManagedClusterSetBinding resources for the deployment in the
   `argocd` namespace and can be viewed using:

   ```
   kubectl get applicationset,placement -n argocd -l app=deployment-rbd
   ```

1. Inspect the ApplicationSet generated applications and resources on the ManagedCluster selected by the Placement.

   The ApplicationSet generates ArgoCD Applications that can be viewed on the hub using:

   ```
   kubectl get applications -n argocd -l app=deployment-rbd
   ```

   The busybox deployment Placement `status` can be viewed on the hub
   using:

   ```
   kubectl get placement deployment-rbd-appset -n argocd
   ```

   The Busybox deployment resources, like the pod and the PVC
   can be viewed on the ManagedCluster using (example ManagedCluster
   `dr1`):

   ```
   kubectl get pod,pvc -n deployment-rbd --context dr1
   ```

## Undeploying a sample application

To undeploy an application delete the applicationset overlay used to
deploy the application:

```
kubectl delete -k applicationset/deployment-k8s-regional-rbd
```

## Enable DR for a deployed application

1. Change the Placement to be reconciled by Ramen

   ```
   kubectl annotate placement deployment-rbd-appset -n argocd \
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
kubectl get placementdecisions -n argocd --context hub \
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
kubectl annotate placement deployment-rbd-appset -n argocd \
    cluster.open-cluster-management.io/experimental-scheduling-disable-
```

At this point the application is managed again by *OCM*.
