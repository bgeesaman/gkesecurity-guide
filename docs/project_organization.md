# Project Organization

## Project and Environment Separation

Using the [Hipster Shop](https://github.com/GoogleCloudPlatform/microservices-demo) as an example workload of eleven coordinating microservices that form a "service offering", there are a few different approaches for how to organize GCP projects and GKE clusters.  Each has pros and cons from a security perspective.

1. **Single GCP Project, Single GKE Cluster, Namespaces per Environment** - A single `project` named `my-hipster-shop` with a GKE `cluster` named `gke-hipster-shop` and three Kubernetes `namespaces`: `hipster-shop-dev`, `hipster-shop-test`, and `hipster-shop-prod`.
    * Pros
        * Simplest `project` strategy
        * Simplest `network` strategy
        * Least expensive `cluster` strategy
    * Cons
        * Weakest "isolation" strategy
        * Weakest "defense in depth" strategy
        * Weakest "resource contention" strategy

1. **Single GCP Project, Three GKE Clusters** - A single `project` named `my-hipster-shop` with three GKE `clusters` named `gke-hipster-shop-dev`, `gke-hipster-shop-test`, and `gke-hipster-shop-prod`.  A single namespace named `hipster-shop` is in each `cluster`.
    * Pros
        * Good "isolation" strategy
        * Good "defense in depth" strategy
        * Good "resource contention" strategy
    * Cons
        * Most complex `project` strategy
        * Simple `network` strategy
        * Most expensive `cluster` strategy

1. **Three GCP Projects, Single GKE Cluster per Project** - Three `projects` named `my-hipster-shop-dev`, `my-hipster-shop-test`, and `my-hipster-shop-prod`.  In each `project`, a GKE `cluster` named `gke-hipster-shop-<env>`, and a single namespace named `hipster-shop` in each `cluster`.
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
* **Cluster Cost** - While the control plane of GKE is free, the additional GKE worker nodes in 3 clusters vs 1 plus the additional maintenance cost of managing (upgrading, securing, monitoring) more clusters increases the overall cost.
* **Isolation** - Does the configuration leverage harder boundary mechanisms like GCP `projects`, separate GKE `clusters` on separate GCE `instances, or does it rely on softer boundaries like Kubernetes `namespaces`?
* **Defense in Depth** - Should a security incident occur, which strategy serves to reduce the available attack surface by default, make lateral movement more difficult to perform successfully, and make malicious activity easier to identify vs normal activity?
* **Resource Contention** - Are workloads competing for resources on the same `nodes`, `clusters`, or `projects`?  Is it possible for a single workload to over-consume resources such that the `nodes` evict `pods`?  If the `cluster` autoscales to add more `nodes`, does that consume all available `cpu`/`memory`/`disk` quota in the `project` and prevent other `clusters` from having available resources to autoscale if they need to grow?

### Best Practices

* **Standardize Early** - Understand that you will be building tools and processes implicitly and explicitly around your approach, so choose the one that meets your requirements and be consistent across all your GKE deployments.  Automation with infrastructure-as-code tools like Terraform for standardizing configuration and naming conventions of `projects` and `clusters` is strongly encouraged.
* **One Cluster per Project** - While it is the most expensive approach to operate, it offers the best permissions isolation, defense-in-depth, and resource contention strategy by default.  If you are serious about service offering and environment separation and have any compliance requirements, this is the best way to achieve those objectives.
* **One Cluster per Service Offering** - Running one `cluster` per colocated set of services with similar data gravity and redundancy needs is ideal for reducing the "blast radius" of an incident. It might seem convenient to place three or four smaller "production" services into a single GKE `cluster`, but consider the scope of an investigation should a compromise occur.  All workloads in that cluster and all the data they touch would have to be "in scope" for the remediation efforts.

### Resources

* [Preparing GKE Environments for Production](https://cloud.google.com/solutions/prep-kubernetes-engine-for-prod)
* [GCP Cost Calculator](https://cloud.google.com/products/calculator/)
* [GKE Pricing](https://cloud.google.com/kubernetes-engine/pricing)
* [Hipster Shop](https://github.com/GoogleCloudPlatform/microservices-demo)
* [Terraform](https://terraform.io)

## Separating Tenants (TODO)

Define hard and soft tenancy.

Per cluster or per namespace.

K8s API vs Pod Network vs Ingress vs Node Resource isolation

### Best Practices

* Hard tenancy - separate projects/clusters
* Soft tenancy - shared projects/clusters

### Resources

* 

## Project Quotas

GCP `projects` have quotas or limits on how many resources are available for potential use.  This doesn't mean that there is available capacity to fulfill the request, and on rare occasions, a particular zone might not be able to provide another GKE worker right away when your cluster workloads need more capacity and node-pool autoscaling kicks in.

GKE uses several GCE related quotas like `cpu`, `memory`, `disk`, and `gpu` of a particular type, but it also uses lesser known quotas like "Number of secondary ranges per VPC".  The resource related quotas are per region and sometimes per zone, so it's important to monitor your quota usage to avoid quota-capped resource exhaustion scenarios from negatively affecting your application's performance during a scale-up event or from preventing certain types of upgrade scenarios.

### Best Practices

* **Monitor Quota Consumption** - Smaller GKE `clusters` will most likely fit into the `project` quota defaults, but `clusters` of a dozen nodes or more may start to bump into the limits.
* **Request Quota Increases Ahead of Time** - Quota increases can take as much as 48 hrs to be approved, so it's best to plan ahead and ask early.
* **Aim for 110%** - If the single GKE `cluster` in a project uses 10, 32-core `nodes`, the total `cpu` cores needed is 320 or more.  To give enough headroom to perform a "blue/green" cluster upgrade if needed, the `project` quota should be at least 640 `cpu` cores in that region to facilitate that approach.  Following the 110% guideline, this would actually be more like 700 `cpus`.  This allows for two, full-sized clusters to be possible for a short duration while the workloads are migrated between the clusters.
* **GCP APIs have Rate Limits** - If your application makes 100s of requests per second or more to say, GCS or GCR, you may run into rate limits designed to protect overuse of the GCP APIs from a single customer affecting all customers.  If you run into these, you may or may not be able to get them increased.  Consider working with GCP support and implementing a different approach with your application.

### Resources

* [GCE Quotas and Limits](https://cloud.google.com/compute/quotas)
* [Working with GCP Quotas](https://cloud.google.com/docs/quota)
