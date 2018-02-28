# Operating etcd clusters for Kubernetes

etcd is a consistent and highly-available key value store used as Kubernetes’ backing store for all cluster data.

Always have a backup plan for etcd’s data for your Kubernetes cluster. For in-depth information on etcd, see etcd [documentation](https://github.com/coreos/etcd/blob/master/Documentation/docs.md).

## Prerequisites

- Run etcd as a cluster of odd members.
- etcd is a leader-based distributed system. Ensure that the leader periodically send heartbeats on time to all followers to keep the cluster stable.
- Ensure that no resource starvation occurs.
- Performance and stability of the cluster is sensitive to network and disk IO. Any resource starvation can lead to heartbeat timeout, causing instability of the cluster. An unstable etcd indicates that no leader is elected. Under such circumstances, a cluster cannot make any changes to its current state, which implies no new pods can be scheduled.
- Keeping stable etcd clusters is critical to the stability of Kubernetes clusters. Therefore, run etcd clusters on dedicated machines or isolated environments for guaranteed resource requirements.

Resource re

## Starting Kubernetes API server

This section covers starting a Kubernetes API server with an etcd cluster in the deployment.
Single-node etcd cluster

Use a single-node etcd cluster only for testing purpose.

1) Run the following:

```bash
./etcd --client-listen-urls=http://$PRIVATE_IP:2379 --client-advertise-urls=http://$PRIVATE_IP:2379
```
2) Start Kubernetes API server with the flag `--etcd-servers=$PRIVATE_IP:2379.`

Replace PRIVATE_IP with your etcd client IP.

## Multi-node etcd cluster

For durability and high availability, run etcd as a multi-node cluster in production and back it up periodically. A five-member cluster is recommended in production. For more information, see FAQ Documentation.

Configure an etcd cluster either by static member information or by dynamic discovery. For more information on clustering, see etcd Clustering Documentation.

For an example, consider a five-member etcd cluster running with the following client URLs: http://$IP1:2379, http://$IP2:2379, http://$IP3:2379, http://$IP4:2379, and http://$IP5:2379. To start a Kubernetes API server:

Run the following:

```bash
./etcd --client-listen-urls=http://$IP1:2379, http://$IP2:2379, http://$IP3:2379, http://$IP4:2379, http://$IP5:2379 --client-advertise-urls=http://$IP1:2379, http://$IP2:2379, http://$IP3:2379, http://$IP4:2379, http://$IP5:2379
```
Start Kubernetes API servers with the flag `--etcd-servers=$IP1:2379, $IP2:2379, $IP3:2379, $IP4:2379, $IP5:2379.`

Replace IP with your client IP addresses.

## Multi-node etcd cluster with load balancer

To run a load balancing etcd cluster:

    Set up an etcd cluster.
    Configure a load balancer in front of the etcd cluster. For example, let the address of the load balancer be $LB.
    Start Kubernetes API Servers with the flag `--etcd-servers=$LB:2379.`
