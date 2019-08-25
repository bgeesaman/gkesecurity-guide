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

### Control Plane Access

The GCE instances running the control plane components like the `kube-apiserver` and `etcd` are not directly accessible via SSH and the GCE API.  They don't appear in `gcloud` command outputs nor in the GCP console.  The GKE control plane is only accessible via the exposed Kubernetes API on `tcp/443`.  By default, the API is assigned a public IP address and has firewall rules that allow access from `0.0.0.0/0`.  While this is conducive to ease of use and access from anywhere, it might not meet your security requirements.

!!! warning
    Sufficient access to the Kubernetes API equates to `root` on all the worker `nodes`. Protecting the Kubernetes API from access misconfiguration, denial-of-service, and exploitation should be high on a security team's priority list.

There are controls available to improve on the default configuration, and they are covered in the next section. 

### Worker Node Access

Because GKE `nodes` are simply GCE `instances` inside `instance groups`, they are accessible via SSH in accordance with the normal GCE routing and VPC firewall rules.  The standard GCE methods for using `gcloud compute ssh` to gain access to the underlying operating system work as expected.

There are two things to consider when granting SSH access to `nodes`:

1. Does SSH access need to be available publicly?  In many cases, additional firewall source restrictions are useful in limiting the allowed subnets from which SSH access can be initiated.  For instance, from a set of external office IP addresses or from a specific VPC `subnet` designated for management purposes.
1. Several primitive IAM roles like `Owner`, `Editor`, `Dataproc Service Agent`, `Compute OS Admin Login`, `Compute Admin`, and `Compute Instance Admin` include the `compute.instances.osAdminLogin` permission.  Users and service accounts with those permissions and network level access can SSH into the worker `nodes` and gain `root` permissions at the operating system level.

!!! warning
    Gaining `root` access to a GKE worker `node` allows that user to view all `secrets` attached to `pods` running on that `node`.  Those `secrets` may include `tokens` that belong to `service accounts` which have elevated permissions to the Kubernetes API.  With that `token`, that user can access the Kubernetes API as that `service account`.  If that `service account` has permissions to modify Role-Based Access Control or to view all `secrets`, that commonly equates to full cluster access (aka "cluster admin").  As "cluster admin" has full control of the Kubernetes API, that equates to `root` on all worker `nodes`.

## Cluster Settings

For a full list of `cluster` configuration level items, the [google_container_cluster](https://www.terraform.io/docs/providers/google/r/container_cluster.html) terraform provider documentation is a fantastic resource for a list of each setting and its purpose.  The list below aims to cover the availability and security-related items with additional supporting guidance:

* **Location** - The `region` ("us-central1") or `zone` ("us-central1-a") where the `cluster` will be located.  Clusters cannot span multiple `regions`.  If a `region` is specified, the control plane will have an `instance` in each of the three `zones`.  If a `zone` is specified, the control plane will have a single instance in that `zone` only.
* **Node Locations** - The list of one to three `zones` where the `nodes` will be located, and it must be in the same `region`.  Recommend: specifying a `region` to obtain a fault-tolerant control plane.

!!! note "Mix-and-Match"
    If `node locations` is specified, it overrides the default behavior.  It's possible to have a `regional` cluster but then use `node locations` to restrict the `zones` to only one or two instead of the default of three.  It's also possible to create a `zonal` cluster with a single control plane instance but then use `node locations` to select one to three `zones` where the nodes will be located.  Typically, this feature is used to help guide `clusters` into configurations to align with large CPU or GPU quota needs.

* **Addons Config**
    * **HTTP Load Balancing** - Enabled by default, this installs the `ingress-gce` HTTP/S Load Balancing and Ingress Controller.  It can be disabled to allow for installation of a different ingress controller solution or left implemented but unused.  Recommend: enabled.
    * **Kubernetes Dashboard** - Now disabled by default, this installs a Kubernetes dashboard deployment inside the `cluster`.  Historically, the dashboard has been misconfigured and has had a small number of security issues that have led to compromises.  With the GCP console providing all the dashboard features needed, this can and should be disabled.  Clusters that were originally created in the Kubernetes `1.6` era and upgraded to the present may still have this enabled.  Recommend: Disabled.
    * **Network Policy** - Disabled by default, this installs the Calico CNI components which enables the configuration of `networkpolicy` resources in the Kubernetes API that can restrict `pod'-to-`pod` communication.  It is "safe" to enable by default as the default effect of the `networkpolicy` configuration in Kubernetes is "allow all traffic" until a `networkpolicy` is defined. Recommend: Enabled.
    * **Istio** - Disabled by default, this installs the Google-managed deployment of Istio inside the `cluster`.  Istio can provide interesting security-related features like "mutual TLS" encryption for traffic between `pods` in the `mesh`, deep traffic inspection in the form of network traces, and granular role-based permissions for egress network traffic and Layer 7 access controls.  In terms of complexity, Istio adds over 50 custom resource definitions that a security team will need to analyze and understand to have awareness of the security posture of the `cluster`. Recommend: "Disabled" until the features are needed.
* **Database Encryption** - Disabled by default, this enables an integration with Cloud KMS to have the API server access a Cloud KMS key to encrypt and decrypt the contents of `secrets` as they are stored and accessed from `etcd`.  In environments where the control plane nodes are potentially accessible, this makes the compromise of a dedicated `etcd` system or `etcd` backup more difficult to extract the `secrets` in plain-text.  In GKE, the control plane systems are not accessible directly by users or attackers, so this adds marginal benefit aside from satisfying regulatory requirements of "databases must encrypt sensitive records at the row-level". Recommend: "Disabled" unless compliance requirements dictate.
* **Default Max Pods Per Node** - Controls how many `pods` can run on a single node.  The default is the hard-coded maximum of 110, but this can be reduced to 64, 32, 16, or 8 if the workload profile is known.  This causes the `subnet` allocated to each GKE `node` to go from a `/24` down to a `/25`, `/26`, and so on, and it can greatly reduce IP address consumption of precious RFC1918 space.  Recommend: the default of 110 unless specific IP space or capacity needs require smaller.
* **Binary Authorization** - Disabled by default.  This addon enables an admission controller that validates `pod` specs against a policy that defines which container image repositories are allowed and/or validates that container images have a valid PGP signature before allowing them to run inside the cluster.  Can be enabled safely as the default policy is to "allow all" until configured. Recommend: Enabled.
* **Kubernetes Alpha** - Disabled by default.  Do not enable on production clusters as alpha clusters are deleted automatically after 30 days by GKE. Recommend: Disabled.
* **Legacy ABAC** - Disabled by default, this is a legacy permissions mechanism that is largely unused in the Kubernetes community.  The default role-based access control (RBAC) mechanism is enabled by default and is preferred. Recommend: Disabled.
* **Logging Service** - Enabled by default, this installs and manages a daemonset that collects and sends all `pod` logs to Stackdriver for processing, troubleshooting, and analysis/auditing purposes. Recommend: `logging.googleapis.com/kubernetes`
* **Maintenance Policy** - Defines a preferred 4-hour time window (in UTC) for when GKE should perform automatic or security-related upgrades. Recommend: setting a 4-hour UTC window with the least workload impact and overlap with operational on-call rotations.
* **Master Authorized Networks** - By default, the implicit allowed CIDR range is `0.0.0.0/0` for which source IPs can connect to the Kubernetes API.  You may specify up to 50 different CIDR ranges.  This setting can be changed as needed without affecting the lifecycle of the running `cluster`, and it is the most common method for reducing the scope of who can reach the Kubernetes `cluster` API to a smaller set of IPs.  In the event of a vulnerability in the API server, enabling a small list of CIDRs to a known set can reduce the risk of an organization delaying maintenance operations to a time that is less busy or risky. Recommend: Configuring a short list of allowed CIDRs.
* **Monitoring Service** - Enabled by default, this installs and manages a daemonset that collects and sends all metrics to Stackdriver for troubleshooting and analysis.  Recommend: `monitoring.googleapis.com/kubernetes`
* **Pod Security Policy** - Disabled by default, this enables the admission controller used to validate the specifications of `pods` to prevent insecure or "privileged" `pods` from being created that allow trivial escaping to the underlying host as `root` and bypassing other security mechanisms like `networkpolicy`.  Recommend: Enabled with extensive testing OR implementing the same features with an Open Policy Agent/Gatekeeper validating admission controller.
* **Authenticator Groups** - Disabled by default, this enables RBAC to use Google Groups in `RoleBinding` and `ClusterRoleBinding` resources instead of having to specify each `user` or `serviceaccount` individually.  For example, a Gsuite administrator creates a group called `gke-security-groups@mydomain.com` and places groups named `gke-admins@` and `gke-developers@` inside it.  Passing `gke-security-groups@mydomain.com` to this setting allows RBAC to reference/lookup the `gke-admins@mydomain.com` and/or `gke-developers@mydomain.com` when evaluating access in the API to resources. Recommend: Enabled if using Gsuite/Google Groups to support easier to manage RBAC policies.
* **Private Cluster** - 
    * **Master IPv4 CIDR Range** - The RFC1918 subnet that the control plane should pick an IP address from when assigning a private IP. Recommend: a `/28` that does not overlap with any other IP space in use.
    * **Enable Private Endpoint** - "false" by default, setting this to "true" instructs the GKE control plane IP to be selected from the master CIDR range and does not expose a public IP address. Recommend: true
    * **Enable Private Nodes** - "false" by default, setting this to "true" instructs the GKE `nodes` to not be assigned a public IP address. Recommend: true

!!! warning "Private Clusters"
    There is a common misconception that a "private cluster" is one that has worker `nodes` with private IP addresses.  This is only half-correct.  A true "private cluster" is one that has both the control plane and `nodes` using private IP addresses, and both of the above settings are necessary to achieve that improved security posture.  When these two settings are enabled and combined with a small list of master authorized networks, the attack surface of the Kubernetes API can be significantly reduced.

* **Remove Default Node Pool** - By default, GKE deploys a `node pool` to be able to run `pods`.  However, its lifecycle is tied to that of the `cluster` object.  Meaning, certain changes may cause the entire `cluster` to be recreated.  It's recommended that you disable/remove the default `node pool` and explicitly declare `node pools` with the desired settings so that they keep their lifecycles independent from the control plane/cluster.
* **Workload Identity** - Disabled by default, this enables the Workload Identity addon and by specifying which Identity "domain" to use.  Without Workload Identity, `pods` wanting to reach GCP APIs would be granted the credentials attached to the underlying GCE `instance`.  The underlying GCE instance has permissions to send logs, send metrics, read from Cloud Storage/Container Registries, and more.  Historically, the `nodes` were assigned the default compute service account which was assigned `Project Editor` permissions.  This meant that every `pod` could potentially have `Project Editor` access!  With Workload Identity, a daemonset on every worker `node` intercepts all calls for instance credentials to the metadata API.  If the Kubernetes `service account` attached to that `pod` has been granted a specific binding to a Google `service account`, then the dynamic crednetials for that specific Google `service account` will be returned to the `pod`.  In essence, this is a way to map Kubernetes `service accounts` to Google `service accounts` and can remove the need for exporting Google `service account` keys in JSON format and storing them manually inside Kubernetes `secrets`.  Recommend: Enabled.
* **IntraNode Visibility** - Disabled by default, this enables the network traffic on the `pod` network to be viewable by the VPC to be available for capture with VPC flow logs.  Recommend: Disabled unless flow logs from `pod` traffic is needed.

## Node Pool Settings

For a full list of `node` configuration level items, the [google_container_node_pool](https://www.terraform.io/docs/providers/google/r/container_node_pool.html) terraform provider documentation lists each setting and its purpose.  The list below aims to cover the availability and security-related items with additional supporting guidance:

* **Auto Repair** - If enabled, should the `node` validate its own health and destroy/recreate the `node` if it detects a problem.  Generally speaking, this can be safely enabled and avoid getting paged for minor issues that can be self-healed. Recommend: Enabled.
* **Auto Upgrade** - If enabled, will attempt to keep the `node` versions in sync with the control plane version during the maintenance window.  Depending on how the applications running inside the cluster respond to losing a worker `node` or how close to maximum capacity the `cluster` is running, this may or may not affect workload availability.  While all applications should be designed to handle this gracefully, it may require additional testing before it is considered appropriate to enable on your production `clusters`. Recommend: Enabled with proper testing.
* **Node Config**
    * **Image Type** - Default is "COS", or "Container-Optimized OS".  This is a hardened, minimalist distribution of Linux that is designed to run containers securely.  Currently, the stable/default version of "COS" leverages a full Docker installation.  "COS_containerd" is available leverages `containerd` instead.  "Ubuntu" is available for specific needs where full control over the Operating System and kernel modules is required, but it defers many operational and security-related tasks to the customer.  Recommend: "COS" until "COS_containerd" becomes the default.
    * **Metadata** - The mechanism for passing attributes to the GCE Instance Metadata of the underlying GCE `instances`.  Perhaps the most important setting which is now default in GKE `1.12+`, is `disable-legacy-endpoints=true`.  This enables only the `v1` metadata API on the instance and disables the "legacy" API version `v1beta1`.  The legacy API does not require a custom HTTP header of `Metadata-Flavor: Google` to be passed in order to access the metadata API.  Applications running on the GCE `instance` or `pods` on the GKE worker `nodes` vulnerable to Server-Side Request Forgery (SSRF) attacks are therefore much less likely to allow exposure of instance credentials via the Metadata API since they need to also control the ability to send the custom HTTP header for the attack to succeed.  Recommend: `disable-legacy-endpoints=true` is set.
    * **Oauth Scopes** - `oauth scopes` serve to "scope" or "limit" the GCP APIs that the credentials of the attached `service account` can access.  It is the _intersection_ or "overlap" of the permissions granted by the `oauth scopes` and the Service Account that defines the actual access.  If a `service account` attached to the `node pool` is granted `Project Owner` but is only assigned the `https://www.googleapis.com/auth/logging.write` `oauth scope`, then those credentials can only write logs to Stackdriver.  Additionally, if the `oauth scope` was `https://www.googleapis.com/auth/cloud-platform` (an alias for "*" or "any GCP API") but the `service account` was only granted `roles/logging.writer`, then those credentials can still only be used to write logs to Stackdriver.  Avoid granting `cloud-platform` or `compute` `oauth scopes`, especially when paired with `Project Editor` or `Project Owner` or `Compute Admin` IAM roles, or any `pod` can leverage that access!  Recommend: the `oauth scopes` assigned to new clusters by default in GKE `1.12+`.
    * **Sandbox Config (Gvisor)** - Disabled by default, this allows the `nodes` to run `pods` with the `gVisor` sandboxing technology.  This provides much greater isolation of the container and its ability to interact with the host kernel, but certain `pod` features are not supported.  Disk performance of those `pods` will be negatively affected, so additional acceptance testing encouraged. Recommend: enabled on dedicated `node pools` where workloads are running that need additional isolation.
    * **Service Account** - By default, the "Default Compute Service Account" in the `project` is assigned, and this has `Project Editor` bound.  To give each `cluster` and/or `node pool` the ability to use separate and least-privilege permissions, this should be a dedicated `service account` created for each `cluster` or `node pool`, and the minimum permissions assigned to it.  Used in conjunction with Oauth Scopes.  Recommend: create and specify a dedicated `service account` and not the default.
    * **Workload Metadata** - Not set by default (the equivalent of "UNSPECIFIED"), this flag enables either the Metadata Concealment Proxy ("SECURE") or Workload Identity ("GKE_METADATA_SERVER") options.  The Workload Identity feature performs all of the same concealment functionality of the Metadata Concealment Proxy buth with the added ability of mapping KSAs to GSAs for dynamic GCP credential access.  Recommend: "GKE_METADATA_SERVER".

### Resources

* [Hardening GKE Clusters](https://cloud.google.com/kubernetes-engine/docs/how-to/hardening-your-cluster)
* [Private GKE Clusters](https://cloud.google.com/kubernetes-engine/docs/how-to/private-clusters)
* [GCE SSH Access](https://cloud.google.com/compute/docs/instances/connecting-to-instance)
* [GKE Master Authorized Networks](https://cloud.google.com/kubernetes-engine/docs/how-to/authorized-networks)
* [Terraform Cluster Provider](https://www.terraform.io/docs/providers/google/r/container_cluster.html)
* [Terraform Node Pool Provider](https://www.terraform.io/docs/providers/google/r/container_node_pool.html)
* [GKE Pod Security Policy](https://cloud.google.com/kubernetes-engine/docs/how-to/pod-security-policies)
* [Open Policy Agent](https://www.openpolicyagent.org/)
* [Gatekeeper](https://github.com/open-policy-agent/gatekeeper)
* [COS](https://cloud.google.com/kubernetes-engine/docs/concepts/node-images)
* [COS_containerd](https://cloud.google.com/kubernetes-engine/docs/concepts/using-containerd)
* [GKE Metadata Concealment](https://cloud.google.com/kubernetes-engine/docs/how-to/protecting-cluster-metadata)
* [GKE Workload Identity](https://cloud.google.com/kubernetes-engine/docs/how-to/workload-identity)
* [GKE OAuth Scopes](https://cloud.google.com/kubernetes-engine/docs/how-to/access-scopes)
* [gVisor Sandboxing](https://cloud.google.com/kubernetes-engine/docs/how-to/sandbox-pods)
* [VPC/Pod Flow Logs](https://cloud.google.com/kubernetes-engine/docs/how-to/intranode-visibility)
