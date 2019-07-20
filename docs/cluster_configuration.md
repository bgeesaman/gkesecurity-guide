# Cluster Configuration

## Geographical Placement and Cluster Types

GKE `clusters` are by default "zonal" clusters.  That is, a single GCE instance running the control plane components is deployed in the same GCP `zone` as the `node-pool` nodes.  "Regional" GKE `clusters` are deployed across three `zones` in the same region.  Three GCE instances running the control plane components are deployed (one per `zone`) behind a single IP and `load balancer`. The `node-pools` spread evenly across the `zones` with a minimum of one `node` per `zone`.

The key benefits to a "regional" `cluster`:

* The `cluster` control plane can handle a single GCP `zone` failure more gracefully.
* When the control plane is being upgraded, only one instance is down at a time which leaves two remaining instances.  Upgrades on "zonal" clusters means a 4-10 minute downtime of the API/control plane while that takes place where you can't use `kubectl` to interact with your `cluster`.

Since there is no additional charge for running a "regional" GKE `cluster` with a highly-available control plane, why use a "zonal" `cluster`?  The two primary reasons are to save on "cross-zone" network traffic costs and to support specific GPU `node-pool` needs.

!!! note "Persistent Disks"

    The default `StorageClass` defines persistent disks that are "zonal".  You may want to add a new `StorageClass` that makes "regional" persistent disks available for `pod` workloads that get rescheduled on another `node` in a different `zone` to still be able to access them.

### Best Practices

* **Use Regional Clusters** - Unless you have specific needs that force you to use "zonal" `cluster`, using "regional" `clusters` offers the best redundancy and availablility for a minor increase in network traffic costs for the majority of use cases.
* **Offer a Regional Persisten Disk `StorageClass`** - Allows `pods` to attach and access persistent disk volumes regardless of where they are scheduled inside the cluster.  This prevents a `zone` failure from allowing a `pod` to be rescheduled and mount that disk on a `node` in another `zone`.

### Resources

* [GKE Regional Clusters](https://cloud.google.com/kubernetes-engine/docs/concepts/regional-clusters)

## IAM and RBAC (TODO)


## Cluster Access


## Cluster Settings


## Node Pool Settings
