# Cluster Lifecycle

## Infrastructure as Code

Documented Terraform example

## Upgrades and Security Fixes

### Control Plane Upgrades

One of the most important problems that GKE helps solve is maintaining the control plane components for you.  Using `gcloud` or the UI console, upgrading the control plane version is a single API call.  There are a few important concepts to understand with this feature:

* **The version of the control plane can and should be kept updated.** - GKE supports the current "Generally Available" (GA) GKE version back to two older minor revisions.  For instance, if the GA version is `1.13.7-gke19`, then `1.11.x` and `1.12.x` are supported.
* **GKE versions tend to trail Kubernetes OSS slightly.** - Kubernetes OSS releases a minor revision approximately every 90-120 days, and GKE is quick to offer "Alpha" releases with those newest versions, but it may take some time to graduate to "GA" in GKE.
* **Control Plane upgrades of Zonal clusters incurs a few minutes of downtime.** - Upgrading to a newer GKE version causes each control plane instance to be recreated in-place. On `zonal` clusters, there is a few minutes where that single instance is unavailable while it is being recreated.  The data in `etcd` is kept as-is and all workloads that are running on the `node pools` remain in place.  However, access to the API with `kubectl` will be temporarily halted.

    Regional clusters perform a "rolling upgrade" approach to upgrading the three control plane instances, and so the upgrade happens one instance at a time.  This leaves the control plane available on the other instances and should incur no noticable downtime.

    For the sake of a highly-available control plane and the bonus that is no additional cost, it's recommended to run `regional` clusters.  If inter-regional traffic costs are a concern, use the `node locations` setting to use a single `zone` but still keep the `regional` control plane.

### Node Pool Upgrades

Upgrades of `node pools` are handled per-`node pool`, and this gives the operator a lot of control over the behavior.  If `auto-upgrades` are enabled, the `node pools` will be scheduled for "rolling upgrade" style upgrades during the configured `maintenance window` time block.  While it's possible to have `node pools` trail the control plane version by two minor revisions, it's best to not trail more than one.

As `node pools` are where workloads are running, a few points are important to understand:

* **Node Pools are resource boundaries** - If certain workloads are resource-intensive or prone to over-consuming resources in large bursts, it may make sense to dedicate a `node pool` to them to reduce the potential for negatively affecting the performance and availability of other workloads.
* **Each Node Pool adds operational maintenance overhead** - Quite simply, each `node pool` adds more work to the operations team.  Either to ensure is automatically upgraded or to perform the upgrades manually if auto-upgrades are disabled.
* **Separating workloads to different Node Pools can help support security goals** - While not a perfect solution, ensuring that workloads handling certain sensitive data are scheduled on separate `node pools` (using `node taints` and `tolerations` on workloads) can help reduce the chance of compromising `secrets` in a container "escape-to-the-host" situation.

    For example, placing PCI-related workloads on a separate `node pool` from other non-PCI workloads means that a compromise of a non-PCI `pod` that allows access to the underlying `host` will be less likely to have access to `secrets` that the PCI workloads are using.

### Security Bulletins

Another benefit of a managed service like GKE is the reduced operational security burden on having to triage, test, and manage upgrades due to security issues.  The GKE Security Bulletins website and Atom/RSS feed provide a consolidated view of all things security-related in GKE.  It is strongly recommended that teams subscribe to and receive automatic updates from the feed as the information is timely and typically very clear on which actions GKE the service is taking and which actions are left to you.

### Resources

* [GKE Cluster Upgrades](https://cloud.google.com/kubernetes-engine/docs/how-to/upgrading-a-cluster#upgrading_the_master)
* [GKE Node Pool Upgrades](https://cloud.google.com/kubernetes-engine/docs/how-to/upgrading-a-cluster#upgrading_nodes)
* [Automatic Node Upgrades](https://cloud.google.com/kubernetes-engine/docs/how-to/node-auto-upgrades)
* [GKE Security Bulletins](https://cloud.google.com/kubernetes-engine/docs/security-bulletins)
* [GKE Release Notes](https://cloud.google.com/kubernetes-engine/docs/release-notes)

## Scaling

While scaling isn't a purely security-related topic, there are a few security and resource availability concerns with each of the common scaling patterns:

* **Node Sizing** - The resources available to GKE worker `nodes` are shared by all `pods` that are scheduled on them.  While there are solid `limits` on CPU and RAM usage, there are not strong resource isolation mechanisms (yet) for disk usage that lands on the `node`.  It is possible for a single `pod` to consume enough disk space and/or inodes and disrupt the health of the `node`.

    Depending on the workload profile you plan on running in the `cluster`, it might make sense to have more `nodes` of a smaller instance type with the faster `pd-ssd` `disk type` than fewer large instances sharing the slower `pd-hdd` `disk type`, for example.  When workload resource profiles drastically differ, it might make sense to use separate `node pools` or even separate `clusters`.

* **Horizontal Pod Autoscaler** - The built-in capability of scaling `replicas` in a `deployment` based on CPU usage may or may not be granular enough for your needs.  Ensure that the total capacity of the `cluster` (capacity of the max number of `nodes` allowed by autoscaling) is greater than the resources used by the max number of `replicas` in the `deployment`.  Also, consider using "external" metrics from Stackdriver like "number of tasks in a pub/sub queue" or "web requests per pod" to be a more accurate and efficient basis for scaling your workloads.
* **Vertical Pod Autoscaler** - Horizontal Pod Autoscaling adds more `replicas` as needed, but that is assuming the `pod` has properly set resource `requests` and `limits` for CPU and RAM set.  Each `pod` should be configured with the proper `requests` for how much CPU and RAM it uses plus 5-10% at idle for its `requests` setting.  The `limit` should be the maximum CPU and RAM that the `pod` will ever use.

    Having accurate `requests` and `limits` set for every `pod` gives the scheduler the correct information for how to place workloads, increases efficiency of resource usage, and reduces the risk of "noisy neighbor" issues from `pods` that consume more than their set share of resources.

    VPA runs inside the `cluster` and can monitor workloads for configured `requests` and `limits` in comparison with actual usage, and can either "audit" (just make recommendations) or actually modify the `deployments` dynamically.  It's recommended to run VPA in a `cluster` in "audit" mode to help find misconfigured `pods` before they cause `cluster`-wide outages.

* **Node Autoprovisioner** - The GKE cluster-autoscaler adds or removes `nodes` from existing `node pools` based on their available capacity.  If a `deployment` needs more `replicas`, there are not enough running `nodes` to handle them, the `node pool` is configured to autoscale, and there are more `nodes` left before the maximum is reached, the autoscaler will add `nodes` until the workloads are scheduled.

    But what if a `pod` is asking for resources that a new `node` can't handle?  For example, a `pod` that asks for `16` cpu cores but the `nodes` are only `8` core instances?  Or if a `pod` needs a GPU but the `node pool` doesn't attach GPUs?  The node autoprovisioner can be configured to _dynamically manage node pools_ for you to help satisfy these situations.  Instead of waiting for an administrator to add a new `node pool`, the cluster autoscaler can do that for you.

### Resources

* [GKE Cluster Sizing](https://cloud.google.com/solutions/scope-and-size-kubernetes-engine-clusters)
* [GKE Cluster Resizing](https://cloud.google.com/kubernetes-engine/docs/how-to/resizing-a-cluster)
* [Horizontal Pod Autoscaler](https://cloud.google.com/kubernetes-engine/docs/how-to/scaling-apps)
* [Vertical Pod Autoscaler](https://cloud.google.com/kubernetes-engine/docs/concepts/verticalpodautoscaler)
* [GKE Node Auto-Provisioner](https://cloud.google.com/kubernetes-engine/docs/how-to/node-auto-provisioning)
