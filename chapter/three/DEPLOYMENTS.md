# Deployments

A Deployment controller provides declarative updates for Pods and ReplicaSets.

You describe a desired state in a Deployment object, and the Deployment controller changes the actual state to the desired state at a controlled rate. You can define Deployments to create new ReplicaSets, or to remove existing Deployments and adopt all their resources with new Deployments.

> Note: You should not manage ReplicaSets owned by a Deployment.

[Docs](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)