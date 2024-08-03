# Creating a test environment

This page will help you to set up an environment for experimenting with
disaster recovery.

## What you’ll need

- Bare metal or virtual machine with nested virtualization enabled
- 8 CPUs or more
- 20 GiB of free memory
- 100 GiB of free space
- Internet connection
- Linux - tested on *Fedora* 37, 38, and 39
- non-root user with sudo privileges (all instructions are for non-root user)

## Setting up your machine

### Install libvirt

Install the `@virtualization` group - on Fedora you can use:

```
sudo dnf install @virtualization
```

Enable the libvirtd service.

```
sudo systemctl enable libvirtd --now
```

Add yourself to the libvirt group (required for minikube kvm2 driver).

```
sudo usermod -a -G libvirt $(whoami)
```

Logout and login again for the change above to be in effect.

### Install required packages

On Fedora you can use:

```
sudo dnf install git make golang helm podman
```

### Clone *Ramen* source locally

```
git clone https://github.com/RamenDR/ramen.git
```

Enter the `ramen` directory - all the commands in this guide assume you
are in ramen root directory.

```
cd ramen
```

### Create a python virtual environment

To keep the ramen tools separate from your host python, we create a
python virtual environment.

```
make venv
```

To activate the environment use:

```
source venv
```

To exit virtual environment issue command *deactivate*.

### Installing required tools

The drenv tool requires various tool for deploying the testing clusters.

#### minikube

On Fedora you can use:

```
sudo dnf install https://storage.googleapis.com/minikube/releases/latest/minikube-latest.x86_64.rpm
```

Tested with version v1.31.1.

#### kubectl

See [Install and Set Up kubectl on Linux](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/)
for details.

Tested with version v1.30.2.

#### clusteradm

See [Install clusteradm CLI tool](https://open-cluster-management.io/getting-started/installation/start-the-control-plane/#install-clusteradm-cli-tool)
for the details.

Version v0.81 or later is required.

#### subctl

See [Submariner subctl installation](https://submariner.io/operations/deployment/subctl/)
for the details.

Version v0.17.0 or later is required.

#### velero

See [Velero Basic Install](https://velero.io/docs/v1.12/basic-install/)
for the details.

Tested with version v1.12.2.

#### virtctl

```
curl -L -o virtctl https://github.com/kubevirt/kubevirt/releases/download/v1.2.1/virtctl-v1.2.1-linux-amd64
sudo install virtctl /usr/local/bin
rm virtctl
```

#### mc

```
curl -L -o mc https://dl.min.io/client/mc/release/linux-amd64/mc
sudo install mc /usr/local/bin
rm mc
```

For more info see
[MinIO Client Quickstart](https://min.io/docs/minio/linux/reference/minio-mc.html#quickstart)

#### kustomize

```
curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh" | bash
sudo install kustomize /usr/local/bin
rm kustomize
```

For more info see
[kustomize install](https://kubectl.docs.kubernetes.io/installation/kustomize/)

#### argocd

```
curl -L -o argocd https://github.com/argoproj/argo-cd/releases/download/v2.11.3/argocd-linux-amd64
sudo install argocd /usr/local/bin/
rm argocd
```

For more info see [argocd installation](https://argo-cd.readthedocs.io/en/stable/cli_installation/)

## Starting the test environment

Before using the `drenv` tool to start a test environment, you need to
activate the python virtual environment:

```
source venv
```

Available environment files:

- `envs/regional-dr.yaml` - regional dr for testing  workloads using RBD and CephFS storage
- `envs/regional-dr-kubevirt.yaml` - regional dr for testing virtual machines using RBD storage

To start a Regional-DR environment use:

```
(cd test; drenv start envs/regional-dr.yaml)
```

Starting the environment takes 8-15 minutes, depending on your machine
and internet connection.

## Build the ramen operator image

Build the *Ramen* operator container image:

```
make docker-build
```

> [!NOTE]
> Select `docker.io/library/golang:1.21` when prompted.

This builds the image `quay.io/ramendr/ramen-operator:latest`

## Deploy and Configure the ramen operator

To deploy the *Ramen* operator in the test environment:

```
ramenctl deploy test/envs/regional-dr.yaml
ramenctl config test/envs/regional-dr.yaml
```

Your environment is ready!

See [initial-setup](initial-setup.md) to learn how to set it up for
experimenting with disaster recovery.

## Deleting the environment

To stop and delete the minikube clusters use `drenv delete` with the
same environment file used to start the environment:

```
(cd test; drenv delete envs/regional-dr.yaml)
```
