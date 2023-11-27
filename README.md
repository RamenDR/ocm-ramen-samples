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

## The sample applications

This repository provides sample applications that can be deployed on
OpenShift and Kubernetes clusters.

### Samples for OpenShift

Applications:

- **odr/busybox-regional-rbd-deploy**
  - path: `odr/busybox-regioanl-rbd-deploy/app`
  - subscription: `odr/busybox-regioanl-rbd-deploy/sub`
  - dr: `odr/busybox-regioanl-rbd-deploy/dr`
  - name: `busybox-regioanl-rbd-deploy`
  - namespace: `busybox-regioanl-rbd-deploy`

- **odr/busybox-metro-rbd-deploy**
  - path: `odr/busybox-metro-rbd-deploy/app`
  - subscription: `odr/busybox-metro-rbd-deploy/sub`
  - dr: `odr/busybox-metro-rbd-deploy/dr`
  - name: `busybox-metro-rbd-deploy`
  - namespace: `busybox-metro-rbd-deploy`

- **odr/busybox-regional-cephfs-deploy**
  - path: `odr/busybox-regional-cephfs-deploy/app`
  - subscription: `odr/busybox-regional-cephfs-deploy/sub`
  - dr: `odr/busybox-regional-cephfs-deploy/dr`
  - name: `busybox-regional-cephfs-deploy`
  - namespace: `busybox-regional-cephfs-deploy`

#### Using the OpenShift Console

On OpenShift the applications can be deployed using the OpenShift
Console. When creating the application, specify:

- `repo`: The URL of this github repo, or if you forked the repo on
  github, the URL of your fork.
- `branch`: The git branch for testing (typically "main")
- `path`: The path to the application directory
- `name`: application name
- `application namespace`: typically same as application name
- PVC Selector: `appname=busybox`

To enable DR for the application, assign a DRPolicy using the Console.

To disable DR for the application you need to update the application
Placement delete the DRPlacementControl resource manually. See
[Disable DR for the OCM application](#disable-dr-for-the-ocm-application)
for more info.

#### Using the command line

You can deploy, enable DR, disable DR and undeploy using the provided
`sub` and `dr` kustomization in each application directory.

### Samples for Kubernetes

On Kubernetes you deploy the application using the provided
`subscription*` kustomization, and you enable or disable DR using the
provided `dr*` kustomization.

The samples are configured to run on *Ramen* test environment. You may
need to change the storage class name to run on your environment.

Applications:

- **k8s/busybox-regional-rbd-deploy**
  - application: `k8s/busybox-regional-rbd-deploy/app`
  - subscription: `k8s/busybox-regional-rbd-deploy/sub`
  - dr: `k8s/busybox-regional-rbd-deploy/dr`
  - name: `busybox-regional-rbd-deploy`
  - namespace: `busybox-regional-rbd-deploy`

- **k8s/busybox-regional-rbd-sts**
  - application: `k8s/busybox-regional-rbd-sts/app`
  - subscription: `k8s/busybox-regional-rbd-sts/sub`
  - dr: `k8s/busybox-regional-rbd-sts/dr`
  - name: `busybox-regional-rbd-sts`
  - namespace: `busybox-regional-rbd-sts`

- **k8s/busybox-regional-rbd-ds**
  - application: `k8s/busybox-regional-rbd-ds/app`
  - subscription: `k8s/busybox-regional-rbd-ds/sub`
  - dr: `k8s/busybox-regional-rbd-ds/dr`
  - name: `busybox-regional-rbd-ds`
  - namespace: `busybox-regional-rbd-ds`

## Deploying sample applications

> [!IMPORTANT]
> The example uses the `k8s/busybox-regional-rbd-deploy` variant.
> To use other applications replace the path and namespace.

1. Deploy an OCM application subscription on hub:

   ```
   kubectl apply -k k8s/busybox-regional-rbd-deploy/sub
   ```

   This creates the required Channel, Subscription, Placement, and
   ManagedClusterSetBinding resources for the busybox application in the
   `busybox-regional-rbd-deploy` namespace and can be viewed using:

   ```
   kubectl get channel,subscription,placement,managedclusterbinding \
       -n busybox-regional-rbd-deploy
   ```

1. Inspect subscribed resources from the channel created in the same
   namespace on the ManagedCluster selected by the Placement.

   The busybox sample Placement `status` can be viewed on the hub using:

   ```
   kubectl get placement busybox-placement -n busybox-regional-rbd-deploy
   ```

   Busybox subscribed resources, like the pod and the PVC can be viewed
   on the ManagedCluster using (example ManagedCluster `cluster1`):

   ```
   kubectl get pod,pvc -n busybox-regional-rbd-deploy --context cluster1
   ```

## Undeploying the sample application

> [!IMPORTANT]
> The example uses the `k8s/busybox-regional-rbd-deploy` variant.
> To use other applications replace the path and namespace.

To undeploy the application delete the subscription kustomization:

```
kubectl delete -k k8s/busybox-regional-rbd-deploy/sub
```

## Enable DR for the OCM application

> [!IMPORTANT]
> The example uses the `k8s/busybox-regional-rbd-deploy` variant.
> To use other applications replace the path and namespace.

1. Change the Placement to be reconciled by Ramen

   ```
   kubectl annotate placement busybox-placement -n busybox-regional-rbd-deploy \
      cluster.open-cluster-management.io/experimental-scheduling-disable=true
   ```

1. Deploy a DRPlacementControl resource for the OCM application on the
   hub, for example:

   ```
   kubectl apply -k k8s/busybox-regional-rbd-deploy/dr
   ```

   This creates a DRPlacementControl resource for the busybox
   application in the `busybox-regional-rbd-deploy` namespace and can be viewed
   using:

   ```
   kubectl get drpc -n busybox-regional-rbd-deploy
   ```

   At this point the application is managed by *Ramen*.

## Disable DR for the OCM application

> [!IMPORTANT]
> The example uses the `k8s/busybox-regional-rbd-deploy` variant.
> To use other applications replace the path and namespace.

1. Ensure Placement is pointing to the cluster where the workload is
   currently placed to avoid data loss if OCM moves the application to
   another cluster.

   The sample `busybox-placement` does not require any change, but if
   you are using an application created by OCP Console, you may need to
   change the cluster name in the placement.

1. Delete the drpc resource for the OCM application on the hub:

   ```
   kubectl delete -k k8s/busybox-regional-rbd-deploy/dr
   ```

   This deletes the DRPlacementControl resource for the busybox
   application, disabling replication and removing replicated data.

1. Change the Placement to be reconciled by OCM

   ```
   kubectl annotate placement busybox-placement -n busybox-regional-rbd-deploy \
       cluster.open-cluster-management.io/experimental-scheduling-disable-
   ```

   At this point the application is managed again by *OCM*.
