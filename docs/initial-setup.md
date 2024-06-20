# Initial setup

## Clone this git repository to get started:

```
git clone https://github.com/RamenDR/ocm-ramen-samples.git
cd ocm-ramen-samples
```

## Switch kubeconfig to point to the OCM Hub cluster

```
kubectl config use-context hub
```

## Create DRClusters and DRPolicy

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

## Setup the common OCM channel resources on the hub:

```
kubectl apply -k channel
```

This creates a Channel resource in the `ramen-samples` namespace and
can be viewed using:

```
kubectl get channel ramen-gitops -n ramen-samples
```
