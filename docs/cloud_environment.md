# Cloud Environment

## Identity and IAM Hierarchy (TODO)

At the foundation of your GCP Organization is a relationship with your trusted identity provider.  Commonly, this is a Gsuite domain or a Cloud Identity installation.  It's the basis of trust for how users and groups are managed throughout the GCP environments.  It's where you audit logins/logouts, enforce MFA and session duration, and more.

It's important to understand that in a GCP Organization, _where_ a user is granted permissions is just as important as _what_ permissions are granted.  For example, granting a user or service account a binding to an IAM role for `Storage Admin` at the Project level allows that account to have all those permissions for all GCS buckets in that Project.

Consider a somewhat realistic scenario:  That Project has GCS buckets for storing audit logs, VPC flow logs, and firewall logs.  The security team is given `Storage Admin` permissions to manage these.  Also in that same Project, a development team wants to store some data used by their application.  They too, get assigned `Storage Admin` permissions to that Project.  There are a few problems:

* Unless the security team is the one that made the IAM permissions assignment, they might not be aware of the exposure of their security log and event data.  This jeopardizes the integrity of the information if needed for incident response and guaranteeing chain of custody.
* The development team now has access to this sensitive security information.  Even though they may never access it or need to, this violates the principle of least privilege.
* Any time a new GCS bucket is added to this Project, both teams will have access to it automatically.  This can often be undesired behavior and tricky to untangle.

Things get even more interesting when you consider a group of Projects organized into a GCP Folder.  Permissions granted to a Folder "trickle down" to the Folders and Projects inside them.  Because permissions are _additive_, a descendent Project can't block or override those permissions.  Therefore, careful consideration must be given to access granted at each level, and special attention is needed for IAM roles bound to users/groups/service accounts at the Folder or Organization levels.  Some examples to consider:

* If a user is assigned the primitive role `Owner` at the Organization level, they will be `Owner` in every single Project in the entire organization.  The risk of compromise of those specific credentials (and by extension, that user's primary computing devices) is immediately increased to undesired levels.
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
* [GSuite Security](https://gsuite.google.com/security/)
* [Cloud Identity Overview](https://cloud.google.com/identity/docs/concepts/overview)
* [Cloud IAM Overview](https://cloud.google.com/iam/docs/overview)

## Organization, Folder, and Project Structure

Why it's so important early, dictates IAM permission granularity/simplicity, project is the IAM boundary for most permissions types.  Why moving folders/projects isn't straightforward.

### Best Practices

* Early decisions have inertia
* **Decide on Naming Conventions Early** - Names of Projects
* Do not map to organization/team names
* Map to geographic folder
* Map to services/apps folders/projects
* Group by deployment env (sandbox/dev/stage/prod)
* Separate projects for GCR, GCS, SecOps (forseti)

### Resources

* Official GCP docs - Mapping IAM to folders/projects 
* Examples of organization structures

## Cluster Networking

VPC (shared vs peering based on administration model)

### Best Practices

* Do not overlap CIDRs
* Use VPC Aliasing
* Watch limits/quotas

### Resources

* Official GCP docs (shared vpc, vpc peering)
* VPC quotas/limits
* GKE VPC aliasing/subnetting
