# Hands On Training

## Objectives

-  Setup and bootstrap K8s cluster (1 master, 2 nodes, 1 ectd)
-  Deploy a [demo](https://github.com/WesleyCharlesBlake/k8s-demo) (eg nginx)

### Picking the right solution

[Deploying K8s](https://kubernetes.io/docs/setup/pick-right-solution/#local-machine-solutions)


## Pre-requisites:
kubectl needs to be installed
AWS CLI

---
# Installing Other Dependencies

## kubectl

`kubectl` is the CLI tool to manage and operate Kubernetes clusters.  You can install it as follows.

### MacOS
From Homebrew:
```
brew install kubernetes-cli
```

From the [official kubernetes kubectl release](https://kubernetes.io/docs/tasks/tools/install-kubectl/):

```
wget -O kubectl https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/darwin/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl
```

### Linux
From the [official kubernetes kubectl release](https://kubernetes.io/docs/tasks/tools/install-kubectl/):

```
wget -O kubectl https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl
```

[Next Topic: Deploying](KOPS-AWS-HA.md)