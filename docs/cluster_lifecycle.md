# Cluster Lifecycle

## Infrastructure as Code

Creating GKE `clusters` using `gcloud` or the UI console is sufficient for testing purposes, but production-ready deployments should be managed with purpose-built tooling that declaratively set the defined state of infrastructure in code form.  Terraform handles this well and it provides an easy to read example code base for how the security-related features are configured:

```console
provider "google-beta" {
  version = "2.13.0"
  project = "my-project-id"
  region  = "us-central1"
}


resource "google_container_cluster" "my-cluster-name" {
  provider = "google-beta"

  name     = "my-cluster-name"
  project  = "my-project-id"

  // Configures a highly-available control plane spread across three
  // zones in this region
  location = "us-central1"

  // It's best to create these networks and subnets in terraform
  // and then reference them here.
  network    = google_compute_network.network.self_link
  subnetwork = google_compute_subnetwork.subnetwork.self_link

  // Set this to the version desired.  It will become the starting
  // point version for the cluster.  Upgrades of the control plane
  // can be initiated by simply bumping this version and running
  // terraform apply.
  min_master_version = "1.13.7-gke19"

  // Specify the newer Kubernetes logging and monitoring features
  // of the Stackdriver integration.
  // Previously "logging.googleapis.com"
  logging_service    = "logging.googleapis.com/kubernetes"
  monitoring_service = "monitoring.googleapis.com/kubernetes"

  // Do not use the default node pool that GKE provides and instead
  // use the node pool(s) define below explicitly.  GKE will actually
  // provision a 1 node node pool and then remove it before making the
  // node pools below.
  remove_default_node_pool = true
  initial_node_count       = 1

  // This is false by default.  RBAC is enabled by default and preferred. 
  enable_legacy_abac = false

  // Enable the Binary Authorization admission controller to allow
  // this cluster to evaluate a BinAuthZ policy when pods are created
  // to ensure images come from the proper sources.
  enable_binary_authorization = true

  // Defines the kubelet setting for how many pods could potentially
  // run on this node.  Depending on instance type, setting this to
  // 64 would be more appropriate and use half the IP space.
  default_max_pods_per_node = 110

  // Enable transparent "application level" encryption of etcd secrets
  // using a KMS keyring/key.  Can be a software or HSM-backed KMS key
  // for extra compliance requirements.
  database_encryption {
    key_name = "my-existing-kms-key"
    state    = "ENCRYPTED"
  }

  addons_config {
    // Do not deploy the in-cluster K8s dashboard and defer to kubectl
    // and the GCP UI console.
    kubernetes_dashboard {
      disabled = true
    }

    // Enable network policy (Calico) as an addon.
    network_policy_config {
      disabled = false
    }

    // Provide the ability to scale pod replicas based on real-time metrics
    horizontal_pod_autoscaling {
      disabled = false
    }
  }

  // Enable intranode visibility to expose pod-to-pod traffic to the VPC
  // for flow logging potential.  Requires enabling VPC Flow Logging
  // on the subnet first
  enable_intranode_visibility = true

  // Enables the PSP admission controller in the cluster.  DO NOT enable this
  // on an existing cluster without first configuring the necessary pod security
  // policies and RBAC role bindings or you will inhibit pods from running
  // until those are correctly configured.  Recommend success in a test env first.
  pod_security_policy_config {
    enabled = true
  }

  // Enable the VPA addon in the cluster to track actual usage vs requests/limits.
  // Safe to enable at any time.
  vertical_pod_autoscaling {
    enabled = true
  }

  // Configure the workload identity "identity namespace".  Requires additional
  // configuration on the node pool for workload identity to function.
  workload_identity_config {
    identity_namespace = "my-project-id.svc.goog.id"
  }

  // Disable basic authentication and cert-based authentication.
  // Empty fields for username and password are how to "disable" the
  // credentials from being generated.
  master_auth {
    username = ""
    password = ""

    client_certificate_config {
      issue_client_certificate = false
    }
  }

  // Enable network policy configurations via Calico.  Must be configured with
  // the block in the addons section.
  network_policy {
    enabled = true
  }

  // Give GKE a 4 hour window each day in which to perform maintenance operations
  // and required security patches. In UTC.
  maintenance_policy {
    daily_maintenance_window {
      start_time = "TODO"
    }
  }

  // Use VPC Aliasing to improve performance and reduce network hops between nodes and load balancers.  References the secondary ranges specified in the VPC subnet.
  ip_allocation_policy {
    use_ip_aliases                = true
    cluster_secondary_range_name  = google_compute_subnetwork.subnetwork.secondary_ip_range.0.range_name
    services_secondary_range_name = google_compute_subnetwork.subnetwork.secondary_ip_range.1.range_name
  }

  // Specify the list of CIDRs which can access the master's API.  This can be
  // a list of up to 50 CIDRs.  It's basically a control plane firewall rulebase.
  // Nodes automatically/always have access, so these are for users and automation
  // systems.
  master_authorized_networks_config {
    cidr_blocks {
      cidr_block = "10.0.0.0/8"
      display_name = "VPC Subnet"
    }
    cidr_blocks {
      cidr_block   = "192.168.0.0/24"
      display_name = "Admin Subnet"
    }
  }

  // Configure the cluster to have private nodes and private control plane access only
  private_cluster_config {
    // Enables the control plane to not be exposed via public IP and which subnet to
    // use an IP from.
    enable_private_endpoint = true
    master_ipv4_cidr_block  = "172.20.20.0/28"
    // Specifies that all nodes in all node pools for this cluster should not have a
    // public IP automatically assigned.
    enable_private_nodes    = true
  }
}

resource "google_container_node_pool" "my-node-pool" {
  provider  = "google-beta"

  name       = "my-node-pool"
  // Spread nodes in this node pool evenly across the three zones in this region.
  location   = "us-central1"
  cluster    = google_container_cluster.my-cluster-name.name
  // Must be at or below the control plane version.  Bump this field to trigger a
  // rolling node pool upgrade.
  version    = "1.13.7-gke19"
  // Because this is a regional cluster and a regional node pool, this is the
  // number of nodes per-zone to create.  1 will create 3 total nodes.
  node_count = 1

  // Overrides the cluster setting on a per-node-pool basis.
  max_pods_per_node = 110

  // The min and max number of nodes (per-zone) to scale to.  This defines a three
  // to 30 node cluster.
  autoscaling {
    min_node_count = 1
    max_node_count = 10
  }

  // Fix broken nodes automatically and keep them updated with the control plane.
  management {
    auto_repair  = "true"
    auto_upgrade = "true"
  }

  node_config {
    machine_type    = "n1-standard-1"
    // pd-standard is often too slow, so using pd-ssd's is recommended for pods
    // that do any scratch disk operations.
    disk_type       = "pd-ssd" 
    // COS or COS_containerd are ideal here.  Ubuntu if specific kernel features or
    // disk drivers are necessary.
    image_type      = "COS"

    // Use a custom service account for this node pool.  Be sure to grant it
    // a minimal amount of IAM roles and not Project Editor like the default SA.
    service_account = "dedicated-sa-name@my-project-id.iam.gserviceaccount.com"

    // Use the default/minimal oauth scopes to help restrict the permissions to
    // only those needed for GCR and stackdriver logging/monitoring/tracing needs.
    oauth_scopes = [
      "https://www.googleapis.com/auth/devstorage.read_only",
      "https://www.googleapis.com/auth/logging.write",
      "https://www.googleapis.com/auth/monitoring",
      "https://www.googleapis.com/auth/servicecontrol",
      "https://www.googleapis.com/auth/service.management.readonly",
      "https://www.googleapis.com/auth/trace.append"
    ]

    // Enable GKE Sandbox (Gvisor) on this node pool.  This cannot be set on the
    // only node pool in the cluster as system workloads are not all compatible
    // with the restrictions and protections offered by gvisor.
    sandbox_config {
      sandbox_type = "gvisor"
    }

    // Protect node metadata and enable Workload Identity
    // for this node pool.  "SECURE" just protects the metadata.
    // "EXPOSE" or not set allows for cluster takeover.
    // "GKE_METADATA_SERVER" specifies that each pod's requests to the metadata
    // API for credentials should be intercepted and given the specific
    // credentials for that pod only and not the node's.
    workload_metadata_config {
      node_metadata = "GKE_METADATA_SERVER"
    }

    metadata = {
      // Set metadata on the VM to supply more entropy
      google-compute-enable-virtio-rng = "true"
      // Explicitly remove GCE legacy metadata API endpoint to prevent most SSRF
      // bugs in apps running on pods inside the cluster from giving attackers
      // a path to pull the GKE metadata/bootstrapping credentials.
      disable-legacy-endpoints = "true"
    }

  }
}
```

### Resources

* [Terraform](https://terraform.io)
* [Terraform Cluster Provider](https://www.terraform.io/docs/providers/google/r/container_cluster.html)
* [Terraform Node Pool Provider](https://www.terraform.io/docs/providers/google/r/container_node_pool.html)

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
