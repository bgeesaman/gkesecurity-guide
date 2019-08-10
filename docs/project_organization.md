# Project Organization

## Project and Environment Separation

Using the [Hipster Shop](https://github.com/GoogleCloudPlatform/microservices-demo) as an example workload of eleven coordinating microservices that form a "service offering", there are a few different approaches for how to organize GCP `projects` and GKE `clusters`.  Each has pros and cons from a security perspective.

1. **Single GCP Project, Single GKE Cluster, Namespaces per Environment** - A single `project` named `my-hipster-shop` with a GKE `cluster` named `gke-hipster-shop` and three Kubernetes `namespaces`: `hipster-shop-dev`, `hipster-shop-test`, and `hipster-shop-prod`.
    * Pros
        * Simplest `project` strategy
        * Simplest `network` strategy
        * Least expensive `cluster` strategy
    * Cons
        * Weakest "isolation" strategy
        * Weakest "defense in depth" strategy
        * Weakest "resource contention" strategy

1. **Single GCP Project, Three GKE Clusters** - A single `project` named `my-hipster-shop` with three GKE `clusters` named `gke-hipster-shop-dev`, `gke-hipster-shop-test`, and `gke-hipster-shop-prod`.  A single `namespace` named `hipster-shop` is in each `cluster`.
    * Pros
        * Good "isolation" strategy
        * Good "defense in depth" strategy
        * Good "resource contention" strategy
    * Cons
        * Most complex `project` strategy
        * Simple `network` strategy
        * Most expensive `cluster` strategy

1. **Three GCP Projects, Single GKE Cluster per Project** - Three `projects` named `my-hipster-shop-dev`, `my-hipster-shop-test`, and `my-hipster-shop-prod`.  In each `project`, a GKE `cluster` named `gke-hipster-shop-<env>`, and a single `namespace` named `hipster-shop` in each `cluster`.
    * Pros
        * Strongest "isolation" strategy
        * Strongest "defense in depth" strategy
        * Strongest "resource contention" strategy
    * Cons
        * Most complex `project` strategy
        * Most complex `network` strategy
        * Most expensive `cluster` strategy

### Strategy Descriptions

* **Project Complexity** - Although GCP `projects` are "free", this refers to the ongoing maintenance and operational overhead of managing and caring for GCP `projects`.  Managing 5 `projects` vs 15 vs 150 has challenges that require more sophisticated tooling and processes to do well.
* **Network Complexity** - Having all GKE `clusters` on the same shared VPC `network` or in separate VPC `networks` will vary the amount of ongoing maintenance of IP CIDR allocation/consumption, Interconnect/Cloud VPN configuration complexity, and complexity of collecting network related flow logs.
* **Cluster Cost** - While the control plane of GKE is free, the additional GKE worker nodes in three `clusters` vs one plus the additional maintenance cost of managing (upgrading, securing, monitoring) more `clusters` increases the overall cost.
* **Isolation** - Does the configuration leverage harder boundary mechanisms like GCP `projects`, separate GKE `clusters` on separate GCE `instances`, or does it rely on softer boundaries like Kubernetes `namespaces`?
* **Defense in Depth** - Should a security incident occur, which strategy serves to reduce the available attack surface by default, make lateral movement more difficult to perform successfully, and make malicious activity easier to identify vs normal activity?
* **Resource Contention** - Are workloads competing for resources on the same `nodes`, `clusters`, or `projects`?  Is it possible for a single workload to over-consume resources such that the `nodes` evict `pods`?  If the `cluster` autoscales to add more `nodes`, does that consume all available `cpu`/`memory`/`disk` quota in the `project` and prevent other `clusters` from having available resources to autoscale if they need to grow?

### Best Practices

* **Standardize Early** - Understand that you will be building tools and processes implicitly and explicitly around your approach, so choose the one that meets your requirements and be consistent across all your GKE deployments.  Automation with infrastructure-as-code tools like Terraform for standardizing configuration and naming conventions of `projects` and `clusters` is strongly encouraged.
* **One Cluster per Project** - While it is the most expensive approach to operate, it offers the best permissions isolation, defense-in-depth, and resource contention strategy by default.  If you are serious about service offering and environment separation and have any compliance requirements, this is the best way to achieve those objectives.
* **One Cluster per Service Offering** - Running one `cluster` per colocated set of services with similar data gravity and redundancy needs is ideal for reducing the "blast radius" of an incident. It might seem convenient to place three or four smaller "production" services into a single GKE `cluster`, but consider the scope of an investigation should a compromise occur.  All workloads in that `cluster` and all the data they touch would have to be "in scope" for the remediation efforts.

### Resources

* [Preparing GKE Environments for Production](https://cloud.google.com/solutions/prep-kubernetes-engine-for-prod)
* [GCP Cost Calculator](https://cloud.google.com/products/calculator/)
* [GKE Pricing](https://cloud.google.com/kubernetes-engine/pricing)
* [Hipster Shop](https://github.com/GoogleCloudPlatform/microservices-demo)
* [Terraform](https://terraform.io)

## Separating Tenants

The previous section covers the "cluster per project" approaches, and this section attempts to guide you through the "workloads per cluster" decisions. Two important definitions to cover first are [Hard and Soft tenancy](https://docs.google.com/document/d/1PjlsBmZw6Jb3XZeVyZ0781m6PV7-nSUvQrwObkvz7jg/edit) as written by Jessie Frazelle:

* **Soft multi-tenancy** - multiple users _within the same organization_ in the same `cluster`. Soft multi-tenancy could have possible bad actors such as people leaving the company, etc. Users are not thought to be actively malicious since they are within the same organization, but potential for accidents or "evil leaving employees." A large focus of soft multi-tenancy is to prevent accidents.
* **Hard multi-tenancy** - multiple users, from various places, in the same `cluster`.  Hard multi-tenancy means that anyone on the `cluster` is thought to be potentially malicious and therefore should not have access to any other tenants resources.

From experience, building and operating a `cluster` with a hard-tenancy use case in mind is very difficult.  The tools and capabilties are improving in this area, but it requires extreme attention to detail, careful planning, 100% visibility into activity, and near-hyper active monitoring.  For these reasons, your journey with Kubernetes and GKE should first solve for the soft-tenancy use case.  The understanding and lessons learned will overlap nearly 100% if you decide to go for hard-tenancy and will _absolutely_ give you the proper frame of reference to decide if your organization can tackle the added challenges.

!!! warning "There is no such thing as a "single-tenant" `cluster`"
    When it comes to workloads, there are always a minimum of two classes: "System" and "User" workloads.  Workloads that are responsible for the operation of the `cluster` (CNI, log export, metrics export, etc) should be isolated from workloads that run actual applications and vice versa.

In Kubernetes, the default separation between these workload types is likely not sufficient for production needs.  System components run on the same physical resources as user workloads, share a common administrative mechanism, share a common layer 3 network with no default access controls, and often run with higher privileges.  Even in GKE, you will want to take steps to address these concerns.

1. **API Isolation** - Using separate Kubernetes `namespaces` to segment workloads for different purposes when it comes to how those resources interact with the Kubernetes API only.  Service accounts per `namespace` tied to granular RBAC policies are the primary approach.
1. **Network Isolation** - Using Kubernetes `namespaces` as an anchor point, defining which `pods` are allowed to talk with each other explicitly via `NetworkPolicy` objects is the primary approach.  For instance, preventing all ingress traffic from non-`kube-system` `namespaces` with the exception of `udp/53` for `kube-dns`.
1. **Privilege Isolation** - Leveraging well-formed containers running as non-privileged users in combination with `PodSecurityPolicies` to prevent user workloads from being able to access sensitive or privileged resources on the worker node and undermining the security of all workloads.
1. **Resource Isolation** - Using features like `ResourceQuotas` to cap overal cpu/memory/persistent disk resource consumption, resources `requests` and `limits` to ensure `pods` are given the resources they need without overcrowding other workloads, and separating security or performance sensitive workloads on separate `Node Pools`. 

Whether you are a single developer running a couple microservices in one `cluster or` a large `organization` with many teams sharing large `clusters`, these concerns are important and should not be overlooked.  The remainder of this guide will attempt to show you how to implement these features in combination and give you confidence in the decisions you make for your use case.

### Best Practices

* **Tenants per Cluster** - From a security perspective, having a single tenant per `cluster` provides the highest degree of separation among tenants, but it is common to allow multiple workloads from different users/teams of similar trust levels to share a `cluster` for cost and operational efficiency.
* **Multiple Untrusted Tenants in a Single Cluster** - This approach is not generally recommended as the level of effort to sufficiently isolate workloads is high and the risk of a vulnerability or mistake leading to a tenant escape is much higher.
* **Separate the System from the Workloads** - No matter which approach is taken, you should take steps to properly isolate the `pods` and `services` that control and manage your `cluster` from the workloads that operate in them.  The system components can have permissions to GCP resources outside your `cluster`, and it's important that an incident with a user workload can't escape to the system workloads and then escape "outside" the `cluster`.

### Resources

* [Hard Multi-Tenancy in Kubernetes](https://blog.jessfraz.com/post/hard-multi-tenancy-in-kubernetes/)
* [Multi-Tenancy Design Space](https://docs.google.com/document/d/1PjlsBmZw6Jb3XZeVyZ0781m6PV7-nSUvQrwObkvz7jg/edit)
* [GKE Cluster Multi-Tenancy](https://cloud.google.com/kubernetes-engine/docs/concepts/multitenancy-overview)

## Project Quotas

GCP `projects` have quotas or limits on how many resources are available for potential use.  This doesn't mean that there is available capacity to fulfill the request, and on rare occasions, a particular zone might not be able to provide another GKE worker right away when your `cluster` workloads need more capacity and node-pool autoscaling kicks in.

GKE uses several GCE related quotas like `cpu`, `memory`, `disk`, and `gpu` of a particular type, but it also uses lesser known quotas like "Number of secondary ranges per VPC".  The resource related quotas are per region and sometimes per zone, so it's important to monitor your quota usage to avoid quota-capped resource exhaustion scenarios from negatively affecting your application's performance during a scale-up event or from preventing certain types of upgrade scenarios.

### Best Practices

* **Monitor Quota Consumption** - Smaller GKE `clusters` will most likely fit into the `project` quota defaults, but `clusters` of a dozen nodes or more may start to bump into the limits.
* **Request Quota Increases Ahead of Time** - Quota increases can take as much as 48 hrs to be approved, so it's best to plan ahead and ask early.
* **Aim for 110%** - If the single GKE `cluster` in a project uses 10, 32-core `nodes`, the total `cpu` cores needed is 320 or more.  To give enough headroom to perform a "blue/green" `cluster` upgrade if needed (bringing up an identical `cluster` in parallel), the `project` quota should be at least 640 `cpu` cores in that region to facilitate that approach.  Following the 110% guideline, this would actually be more like 700 `cpus`.  This allows for two, full-sized `clusters` to be possible for a short duration while the workloads are migrated between the `clusters`.
* **GCP APIs have Rate Limits** - If your application makes 100s of requests per second or more to say, GCS or GCR, you may run into rate limits designed to protect overuse of the GCP APIs from a single customer affecting all customers.  If you run into these, you may or may not be able to get them increased.  Consider working with GCP support and implementing a different approach with your application.

### Resources

* [GCE Quotas and Limits](https://cloud.google.com/compute/quotas)
* [Working with GCP Quotas](https://cloud.google.com/docs/quota)
