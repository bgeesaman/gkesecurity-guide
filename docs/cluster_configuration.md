# Cluster Configuration

## The GKE Cluster Architecture

Google Kubernetes Engine makes several key architectural decisions for you that may differ from other Kubernetes `cluster` installations.  These choices are made in the interest of ease of use, operational simplicity, and security:

* **There is no direct access to the control plane systems** - GKE manages the GCE instances running `etcd` and the `api server` and supporting components, and does not expose them to you or an attacker via SSH, for example.  In exchange for giving up direct control over all API server configuration flags, your GKE cluster offers very strong protection of the core components and sensitive data in `etcd` automatically.  This means your control over the control plane is restricted to the configuration options on the GKE `cluster` object.
* **Upgrades are handled via GKE Operations** - Upgrades and downgrades are handled via GKE operations through API calls.  GKE versions (which map to Kubernetes versions) and worker `node` OS security and version upgrades are all core features of the built-in upgrade process.
* **The default worker `node` operating system is COS** - Container-optimized OS is a hardened, minimal operating system engineered specifically to run containers and to function as a Kubernetes worker `node`.  While SSH is enabled by default and integrates with GCP SSH functionality, it's goal is to make SSHing into each worker `node` unnecessary by handling patching and upgrades as a part of the `node pool` lifecycle.  There is an Ubuntu worker `node` option for supporting unique use cases, but it requires additional administration in terms of upgrades and security patches.
* **The CNI is not configurable** - The native GKE CNI is used to connect worker `nodes` and `pods` to the GCP Network and is not currently replacable.
* **Network Policy enforcement is done via Calico** - Enabling Network Policy enforcement installs a Calico deployment and daemonset as a managed addon for you.
* **Logging and Monitoring is done via Stackdriver** - By default, all logs and performance metrics are sent to Stackdriver in the current GCP `project`.

### Resources

* [GKE Architecture](https://cloud.google.com/kubernetes-engine/docs/concepts/cluster-architecture)
* [GKE Upgrades](https://cloud.google.com/kubernetes-engine/docs/how-to/upgrading-a-cluster)
* [Container-Optimized OS](https://cloud.google.com/container-optimized-os/docs/concepts/features-and-benefits)
* [GKE CNI](https://cloud.google.com/kubernetes-engine/docs/concepts/network-overview)
* [GKE Stackdriver](https://cloud.google.com/monitoring/kubernetes-engine/)

## Geographical Placement and Cluster Types

GKE `clusters` are by default "zonal" clusters.  That is, a single GCE instance running the control plane components is deployed in the same GCP `zone` as the `node-pool` nodes.  "Regional" GKE `clusters` are deployed across three `zones` in the same region.  Three GCE instances running the control plane components are deployed (one per `zone`) behind a single IP and `load balancer`. The `node-pools` spread evenly across the `zones` with a minimum of one `node` per `zone`.

The key benefits to a "regional" `cluster`:

* The `cluster` control plane can handle a single GCP `zone` failure more gracefully.
* When the control plane is being upgraded, only one instance is down at a time which leaves two remaining instances.  Upgrades on "zonal" clusters means a 4-10 minute downtime of the API/control plane while that takes place where you can't use `kubectl` to interact with your `cluster`.

Since there is no additional charge for running a "regional" GKE `cluster` with a highly-available control plane, why use a "zonal" `cluster`?  The two primary reasons are to save on "cross-zone" network traffic costs and to support specific GPU `node-pool` needs.

!!! note "Persistent Disks"

    The default `StorageClass` defines persistent disks that are "zonal".  You may want to add a new `StorageClass` that makes "regional" persistent disks available for `pod` workloads that get rescheduled on another `node` in a different `zone` to still be able to access them.

### Best Practices

* **Use Regional Clusters** - Unless you have specific needs that force you to use a "zonal" `cluster`, using "regional" `clusters` offers the best redundancy and availablility for a minor increase in network traffic costs for the majority of use cases.
* **Offer a Regional Persistent Disk `StorageClass`** - Allows `pods` to attach and access persistent disk volumes regardless of where they are scheduled inside the cluster.  This prevents a `zone` failure from allowing a `pod` to be rescheduled and mount that disk on a `node` in another `zone`.

### Resources

* [GKE Regional Clusters](https://cloud.google.com/kubernetes-engine/docs/concepts/regional-clusters)

## Google Cloud IAM and Kubernetes RBAC

* **Google Cloud Identity and Access Management (IAM)** - The system in GCP that grants permissions via GCP `IAM Roles` to users and service accounts to access GCP APIs.
* **Kubernetes Role-Based Access Control (RBAC)** - The native system inside Kubernetes that grants permissions via Kubernetes `roles` and `clusterroles` to users and service accounts to access the Kubernetes API Server.

The two APIs that can be used to interact with the GKE service and a GKE cluster are:

1. **GCP GKE API (container.googleapis.com)** - Used to create/update/delete the `cluster` and `node pools` that comprise the GKE `cluster` and to obtain connection and credential information for how to access a given `cluster`.
2. **Kubernetes API of the `cluster`** - The unique Kubernetes API Server endpoint running on the GKE Control Plane systems for a specific `cluster`.  It controls resources and access to resources running _inside_ the cluster.

This means that there are two main categories of permissions that you have to consider:

* **Cluster Administration** - Permissions associated with administering the `cluster` itself.
* **Cluster Usage** - Permissions associated with granting what is allowed to run inside the cluster.

There are a couple key points to understand about how Cloud IAM and Kubernetes RBAC can be used to grant permissions in GKE:

1. GCP Cloud IAM is administered at the `project` level, and the granted permissions apply to all GKE `clusters` in the `project`. They remain in-place even if a `cluster` is deleted from the `project`.
2. Kubernetes RBAC permissions are administered per-`cluster` and are stored inside each `cluster`.  When that `cluster` is destroyed, those permissions are also destroyed.
3. GCP Cloud IAM permissions have no concept of Kubernetes `namespaces`, so the permissions granted apply for all `namespaces`.
4. Kubernetes RBAC can be used to grant permissions to access resources in all `namespaces` or only in specific `namespaces`.
5. Kubernetes RBAC cannot be used to grant permissions to the GCP GKE API (container.googleapis.com).  This is performed solely by Cloud IAM.
6. Both systems are _additive_ in that they only grant permissions.  They cannot remove or negate permissions from themselves or each other.

!!! note "IAM and RBAC _combine_ inside the `cluster`"

    When accessing a resource (e.g. list pods in the default namespace) via the Kubernetes API of a GKE `cluster`, it can be granted via IAM _or_ RBAC.  They are effectively combined.  If either grants access, the request is permitted.

### Best Practices

* There are several predefined IAM `roles` for GKE `clusters` that can ease administration as you are getting started with GKE.  However, the permissions they grant are often overly broad and violate the priciple of least privileges.  You will want to create custom IAM `roles` with just the permissions needed, but remember that IAM permissions apply to all `namespaces` and cannot be limited to a single `namespace`.

* To maintain least privilege, it's recommended to leverage a minimal IAM `role` to gain access to the GKE API and then use Kubernetes RBAC `roles` and `clusterroles` to define what resources they can access inside the `cluster`.

    In order for `users` and `service accounts` to perform a `gcloud container clusters get-credentials` call to generate a valid `kubeconfig` for a GKE `cluster`, they need to have the following permissions:

    * `container.apiServices.get`
    * `container.apiServices.list`
    * `container.clusters.get`
    * `container.clusters.list`
    * `container.clusters.getCredentials`

    If these permissions are grouped into a custom IAM `role`, that IAM `role` can be conveniently bound to a Gsuite/Cloud Identity `group` which includes all users that need access to the `cluster`.

    From this point, the `users` and `service accounts` can be granted access to resources as needed with `cluster`-wide or per-`namespace` granularity.  This has the benefit of minimizing IAM changes needed, ensuring access granted is per-`cluster`, giving the highest granularity, and making troubleshooting permissions an RBAC-only process.

* With the above approach of deferring nearly all permissions to in-`cluster` RBAC instead of IAM, there is one exception: assigning the `Kubernetes Engine Admin` predefined IAM `role` at the `project` or `folder` level to a small number of trusted administrators to ensure that they don't accidentally remove their own access via an RBAC configuration mistake.

### Resources

* [GKE IAM and RBAC](https://cloud.google.com/kubernetes-engine/docs/concepts/access-control)
* [Predefined GKE Roles](https://cloud.google.com/kubernetes-engine/docs/how-to/iam#predefined)
* [GKE RBAC](https://cloud.google.com/kubernetes-engine/docs/how-to/role-based-access-control)

## Cluster Access

Private vs Public CP and Nodes

SSHing to nodes

IAP and Bastion


## Cluster Settings


## Node Pool Settings
