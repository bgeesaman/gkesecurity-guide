# Project Organization

## Project Quotas (TODO)

CPU/RAM/GPU/Disk, Subnets, Instances per mig

### Best Practices

* 110% (more than double) current usage
* Subnets/VPC

### Resources

* Official GCP Docs - Quotas and limits

## Project and Environment Separation

Three types of deployment models, pros and cons, easy of switching between them, clusters per project

### Best Practices

* 1 cluster per project
* 1 cluster per colocated set of services with similar data gravity and redundancy needs

### Resources

* Official GKE Docs?
* asdf

## Separating Tenants

Define hard and soft tenancy. Per cluster or per namespace

### Best Practices

* Hard tenancy - separate projects/clusters
* Soft tenancy - shared projects/clusters

### Resources

* ?
