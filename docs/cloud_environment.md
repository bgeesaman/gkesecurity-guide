# Cloud Environment

## Identity and IAM Hierarchy

At the foundation of your Google Cloud Platform (GCP) `organization` is a relationship with your trusted identity provider.  Commonly, this is a Gsuite domain or a Cloud Identity installation.  It's the basis of trust for how users and groups are managed throughout the GCP environments.  It's where you audit logins/logouts, enforce MFA and session duration, and more.

It's important to understand that in a GCP `organization`, _where_ a user is granted permissions is just as important as _what_ permissions are granted.  For example, granting a user or service account a binding to an IAM role for `Storage Admin` at the `project` level allows that account to have all those permissions for all Google Cloud Storage (GCS) buckets in that `project`.

Consider a somewhat realistic scenario:  That `project` has GCS buckets for storing audit logs, VPC flow logs, and firewall logs.  The security team is given `Storage Admin` permissions to manage these.  Also in that same `project`, a development team wants to store some data used by their application.  They too, get assigned `Storage Admin` permissions to that `project`.  There are a few problems:

* Unless the security team is the one that made the IAM permissions assignment, they might not be aware of the exposure of their security log and event data.  This jeopardizes the integrity of the information if needed for incident response and guaranteeing chain of custody.
* The development team now has access to this sensitive security information.  Even though they may never access it or need to, this violates the principle of least privilege.
* Any time a new GCS bucket is added to this `project`, both teams will have access to it automatically.  This can often be undesired behavior and tricky to untangle.

Things get even more interesting when you consider a group of `projects` organized into a GCP Folder.  Permissions granted to a Folder "trickle down" to the Folders and `projects` inside them.  Because permissions are _additive_, a descendent `project` can't block or override those permissions.  Therefore, careful consideration must be given to access granted at each level, and special attention is needed for IAM roles bound to users/groups/service accounts at the Folder or Organization levels.  Some examples to consider:

* If a user is assigned the primitive role `Owner` at the Organization level, they will be `Owner` in every single `project` in the entire organization.  The risk of compromise of those specific credentials (and by extension, that user's primary computing devices) is immediately increased to undesired levels.
* Granting `Viewer` at the Organization level allows reading all GCS buckets, all StackDriver logs, and viewing the configurations of most resources.  In many cases, sensitive data is found in any or all of those places that can be used to escalate privileges.

!!! warning "IAM "Overcrowding""

    The IAM page in the GCP Console tends to get "crowded" and harder to visually reason about when there are lots of individual bindings.  Each IAM Role binding increases the cognitive load when reviewing permissions.  Over time, this can get unwieldy and difficult to consolidate once systems and services are running in production.

### Best Practices

* **Use a single Identity Provider** - If all users and groups are originating from a single domain and are managed centrally, maintaining the proper IAM permissions throughout the GCP Organization is greatly simplified.
* **Ensure all accounts use Multi-Factor Authentication (MFA)** - Protecting the theft and misuse of GCP account credentials is the first and most important step to protecting your GCP infrastructure.  If a set of credentials is stolen or a laptop lost, the window for those credentials to be valid is greatly reduced.
* **Consolidate Users into Groups** - Instead of granting individual users access, consider using a group, placing that user in that group, and granting access to the group.  This way, if that person leaves the organization, it doesn't leave orphaned/unused accounts clogging up the IAM permissions listings that have to be manually removed using gcloud/UI.  Also, when replacing that person who left, it's a simpler process of just adding the new person(s) to the same groups.  No gcloud/UI changes are then necessary.
* **Follow the Principle of Least Privilege** - Take care when granting access Project-wide and even greater care when granting access at the Folder and Organization levels.  There are often unintended consequences of over-granting permissions.  Additionally, it's much harder to safely remove that binding as time goes on because inherited permissions might be required in a descendent project.

### Resources

* [Creating and Managing GCP Organizations](https://cloud.google.com/resource-manager/docs/creating-managing-organization)
* [GCP Resource Hierarchy](https://cloud.google.com/resource-manager/docs/cloud-platform-resource-hierarchy)
* [GSuite Security](https://gsuite.google.com/security/)
* [Cloud Identity Overview](https://cloud.google.com/identity/docs/concepts/overview)
* [Cloud IAM Overview](https://cloud.google.com/iam/docs/overview)

## Organization, Folder, and Project Structure

Resource hierarchy decisions made early on in your GCP journey tend to have a "inertia".  Meaning, once they are configured a certain way, they become entrenched and harder to change as time goes on.  If configured well, they can reduce administrative and security toil in managing resources and access to those resources in a clean, manageable way.  If configured sub-optimally, you might feel like making administrative changes is overly complex and burdensome.  Often, these decisions are often made in haste during design and implementation efforts "just to get things working".  It might seem that there is pressure to get things perfect from the outset, but there is some level of built-in flexibility that you can take advantage of -- provided certain steps are taken.

The two components of the GCP resource hierarchy that are trickiest to change are the `organization` and the non-empty `project`.  The `organization` is the root of all resources, so it establishes the hierarchy.  `Projects` cannot be renamed, and resources inside `projects` cannot be moved outside of `projects`.  `Folders` can be made up to three levels deep, and `projects` can be moved around inside `folders` as needed.  
What this means:

* Permissions assigned at the `organization` level descend to everything below regardless of `folder` and `project` structure.
* Renaming `projects` or moving resources out of a `project` requires migration of resources, so careful thought to their name and what runs inside them is needed.
* `Folders` are a powerful tool in designing and adapting your IAM permissons hierarchy as needs change.

Consider the following diagram of a GCP Organization:

![Policy Hierarchy Diagram](img/policy-hierarchy.png)

The `organization` is named `example.com`, and that is the root of the IAM hierarchy.  Skipping over the `folders` tier for a second -- the `projects` are organized by operational environment which is very common and a best practice for keeping resources completely separated.  Resources in the `example-dev` `project` have no access by default to `example-test` or `example-prod`.  "Fantastic!" you might say, and copy this approach.

However, depending on your organization's needs, this hierarchy might not be ideal!  Let's break down the potential issues with the `folders` and the resources inside each `project`:

1. In the `example-dev` project, `instance_a` and `service_a` may or may not be part of the same overall "service".  If they both operate on data in `bucket_a`, for example, this makes sense.  But if `instance_a` is managed by a separate team and shares nothing with `service_a` or `bucket_a`, you have a permissions scoping and "attack blast radius" problem.  Permissions assigned on the `example-dev` project might be shared in undesirable ways between `instance_a` and `service_a`.  In the event of a security compromise, attackers will be much more likely to obtain credentials with shared permissions and access nearby resources inside that same project. Finally, unless additional measures are taken with custom log sinks, the logs and metrics in Stackdriver will be going to the same project.  Administrators and developers of the `instance_a` and `service_a` would be able to see each other's logs and metrics.  
1. In the `example-test` project, two compute instances and two GCS buckets are in place.  If `bucket_b` and `bucket_c` are part of the same overall service, this makes sense.  But if `bucket_b` serves data with PII in it and `bucket_c` is where VPC flow logs are stored, this increases the chance of accidental exposure if the `Storage Viewer` IAM Role is granted at the project level.
1. At the `folder` and `project` level, it may seem reasonable to name the folder after the team name.  e.g. `team-a-app1-dev`. However, in practice, team names and organization charts in the real world tend to change more often than GCP resources.  To remain flexible, it's much easier to use GCP groups to combine users into teams and then assign IAM permissions for that group at the desired level.  Otherwise, you'll end up having `team_a` managing the `team_b` projects which will get very confusing.
1. If your organization becomes large and has lots of geographic regions, a common practice is to have a top-level `folder` for regions like `US`, `EU`, etc.  While this places no technical boundary on where the resources are deployed in descendent projects, it may make compliance auditing more straightforward where data must remain in certain regions or countries.
1. Certain services, like GCS and Google Container Registry (GCR), don't have very granular IAM permissions available to them unless you want to maintain per-object ACLs.  To avoid some of that additional overhead, a common practice is to make dedicated GCP `projects` just for certain types of common GCS buckets.  Similarly, with GCR, container image push/pull permissions are coarse GCS "storage" permissions.  Dedicated `projects` are therefore a near requirement for separate GCR registries.
1. Highly privileged administrative components that have organization-wide permissions should always be in dedicated `projects` near the top of the `organization`.  A common example is the `project` where [Forseti Security](https://forsetisecurity.org/) components are installed.  If this project was nested inside several levels of `folders`, an IAM Role granted higher in that structure would unintentially provide access to those systems.

### Best Practices

* **Certain Decisions Have inertia** - Some decisions like organization domain name and project names carry a lot of inertia and are hard to change without migrating resources, so it's important to decide on a naming convention for `projects` early and stick with it.
* **Consider Central Control of Project Creation** - If project creation is to be controlled centrally, you can use an Organization Access Policy to set which group has the `Project Creator` IAM Role.
* **Avoid Embedding Team Names in GCP Resource Names** - Map `folder` and `project` names more closely with service offerings and operational environments.  Use groups to establish the concept of "teams" and assign groups the IAM permissions to the `folders` and `projects` they need.
* **Use Folder Names at the Top Level Carefully** - Consider using high level `folders` to group sub-`folders` and `projects` along boundaries that shouldn't change often but help guide organization and auditing.  e.g. Geographic region.
* Use separate `projects` for each operational environment.  It's common to append `-dev`, `-test`, and `-prod` to project names and then group them into a `folder` named for that `service`.  e.g. The folder `b2b-crm` holds the `projects` named `b2b-crm-dev`, `b2b-crm-test`, and `b2b-crm-prod`.

### Resources

* [Using Resource Hierarchy for Access Control](https://cloud.google.com/iam/docs/resource-hierarchy-access-control)
* [Using IAM Securely](https://cloud.google.com/iam/docs/using-iam-securely)
* [Organization Access Policy](https://cloud.google.com/resource-manager/docs/access-control-org)
* [Forseti Security](https://forsetisecurity.org/)

## Cluster Networking (TODO)

VPC (shared vs peering based on administration model)

### Best Practices

* Do not overlap CIDRs
* Use VPC Aliasing
* Watch limits/quotas

### Resources

* Official GCP docs (shared vpc, vpc peering)
* VPC quotas/limits
* GKE VPC aliasing/subnetting
