# Cluster Observability

!!! note Stackdriver Kubernetes Engine Logging and Monitoring

    This section only refers to GKE Clusters using the newer "Stackdriver Kubernetes Engine Logging" and "Stackdriver Kubernetes Engine Monitoring" mechanism.  If you are using Terraform, the value for the `logging_service` should be `logging.googleapis.com/kubernetes` and `monitoring_service` should be `monitoring.googleapis.com/kubernetes`.

## Logging

### GKE Logs

When `logging.googleapis.com/kubernetes` is enabled for a `cluster` in a `project`, the following `resource_type`s now appear in Stackdriver:

* **GKE Cluster Operations** - `resource.type="gke_cluster"` Captures actions like `CreateCluster` and `DeleteCluster` similar to what is shown via `gcloud container operations list`. Use this to know when `clusters` are created/updated/deleted (including the full object specification details/settings), who performed that action, if it was authorized, and more.  Add `protoPayload.authorizationInfo.granted!=true` to the filter to see all unauthorized attempts. 
* **GKE NodePool Operations** - `resource.type="gke_nodepool"` Similar to GKE Cluster Operations but specific to operations on `node pools`.  `gcloud container operations list` mixes both `cluster` and `node pool` operations into a single output for ease of monitoring upgrades from one gcloud command.
* **Kubernetes Cluster** - `resource.type="k8s_cluster"` Captures actions against the Kubernetes API Server/Control Plane of all `clusters` in the `project`.  Add the filter `resource.labels.cluster_name="standard-cluster-2" resource.labels.location="us-central1-a"` to limit the logs to a specific `cluster` in a specific `location` (region or zone).  Use this to know when Kubernetes resources are created/updated/deleted (including the full object specification details), who performed that action, if it was authorized, and more.  Add `labels."authorization.k8s.io/decision"="forbid"` or `"allow"` to the filter to see which actions were permitted by RBAC inside the cluster.  When paired with `protoPayload.authenticationInfo.principalEmail="myemail@domain.com"`, this filter can help with troubleshooting RBAC failures during the development of RBAC `ClusterRoles` and `Roles`.

    You might want to configure filters and sinks/exports for all RBAC failures and look for anomalies/failures of service accounts.  For instance, if `protoPayload.authenticationInfo.principalEmail="kubelet"` does not have a UserAgent like: `protoPayload.requestMetadata.callerSuppliedUserAgent="kubelet/v1.13.7 (linux/amd64) kubernetes/7d3d6f1"`, but instead one corresponding to `curl` or `kubectl`, it's a near-guarantee that a malicious actor has stolen and is using the `kubelet`'s credentials.

* **Kubernetes Node** - `resource.type="k8s_node"` Captures all logs exported from the GCE Instance operating system for all `nodes` in all `clusters` in the `project`.  In the case of `COS`, these are `systemd` logs.  Add the filter `jsonPayload._SYSTEMD_UNIT="kubelet.service"` to see logs from the `kubelet` systemd unit, and add a filter like `resource.labels.node_name="gke-standard-cluster-1-default-pool-de92f9ef-qwpq"` to limit the scope to a single `node`.  
* **Kubernetes Pod** - `resource.type="k8s_pod"` Captures `pod` "events" logs that describe the lifecycle/operations of `pods`.  If you run `kubectl describe pod <podname>` and view the "Events" section manually during troubleshooting, these will be familiar.  It's important to note that the "Events" logs in Kubernetes do not persist after a certain period of time, so shipping these to Stackdriver allows for diagnosing issues after the fact.  Adding the filter `resource.labels.pod_name="mynginx-5b97b974b6-6x589" resource.labels.cluster_name="standard-cluster-2" resource.labels.location="us-central1-a" resource.labels.namespace_name="default"` allows for narrowing things down to a specific `pod` in a specific `namespace` in a specific `cluster`.
* **Kubernetes Container** - `resource.type="k8s_container"` Captures the per-`container` logs that your application emits.  The same output as if you run `kubectl logs mypod -c mycontainer`.  In addition to the `cluster`/`namespace`/`pod` filters also used by the `pod` logs, add `resource.labels.container_name="mycontainer"` to get the logs from a specific `container`.

### GCE Logs for GKE Clusters

* **GCE Instance Group** - `resource.type=""`
* **GCE Instance Group Manager** - `resource.type=""`
* **GCE Instance Template** - `resource.type=""`
* **GCE VM Instance** - `resource.type=""`
* **GCE Disk** - `resource.type=""`

### Resources

* [GKE Audit Logging](https://cloud.google.com/kubernetes-engine/docs/how-to/audit-logging)

## Monitoring

### Node Metrics

### Kubernetes State Metrics

### Workload Metrics
