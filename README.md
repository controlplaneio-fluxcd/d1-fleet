# d1-fleet

This repository is managed by the platform team who are responsible for
the Kubernetes infrastructure and have direct access to the fleet of clusters.

## Scope

- Bootstrap Flux with multi-tenancy restrictions on fleet clusters
- Configure the delivery of platform components (defined in `d1-infra` repository) 
- Configure the delivery of applications (defined in `d1-apps` repository)

## Access Control

The platform team that manages this repository must have the following access rights:

- **Admin** rights to the `d1-fleet` repository
- **Cluster Admin** rights to all clusters in the fleet

