# d1-fleet

This repository is managed by the platform team who are responsible for
the Kubernetes infrastructure and have direct access to the fleet of clusters.

## Scope and Access Control

The platform team that manages this repository must have **admin** rights to the `d1-fleet` repository
and **cluster admin** rights to all clusters in the fleet to be able to perform the following tasks:

- Bootstrap Flux with multi-tenancy restrictions on fleet clusters.
- Configure the delivery of platform components (defined in `d1-infra` repository).
- Configure the delivery of applications (defined in `d1-apps` repository).

## Create a GitHub Account for Flux

Create a new GitHub account for the Flux bot. This account will be used by the Flux CLI and
the Flux controllers running on clusters to authenticate with GitHub.

Create a GitHub team under your organisation for the bot account and give it the following permissions:

- Read and write access to the `d1-fleet` repository (required for cluster bootstrap)
- Push access to the `main` branch of the `d1-fleet` repository (required for cluster bootstrap)
- Read and write access to the `d1-infra` and `d1-apps` repositories (required for cluster reconciliation and image automation)

Create a GitHub fine-grained personal access token for the bot account with
the following permissions for the Flux repositories:

- `Administration` -> `Access: Read-only`
- `Commit statuses` -> `Access: Read and write`
- `Contents` -> `Access: Read and write`
- `Metadata` -> `Access: Read-only`

## Bootstrap Procedure

The bootstrap procedure is a one-time operation that sets up the Flux controllers on the cluster, and
configures the delivery of platform components and applications.

After bootstrap, Flux will monitor the repository for changes and will reconcile itself from
the Kubernetes manifests pushed by the Flux CLI in the repository. Changes to Flux configuration
and version upgrades are done by modifying the repository and letting Flux reconcile the changes,
there is no need to run the bootstrap command again nor connect to the cluster.

### Bootstrap the staging cluster

Make sure to set the default context in your kubeconfig to your staging cluster, then run bootstrap with:

```shell
export GITHUB_TOKEN=<Flux Bot PAT>

flux bootstrap github \
  --registry=ghcr.io/fluxcd \
  --components-extra=image-reflector-controller,image-automation-controller \
  --owner=controlplaneio-fluxcd \
  --repository=d1-fleet \
  --branch=main \
  --token-auth \
  --path=clusters/staging
```

The Flux CLI will use the bot PAT to push two commits to the `d1-fleet` repository:

- First commit to create the `clusters/staging/flux-system/gotk-components.yaml` file
  which contains the flux-system namespace, RBAC, network policies, CRDs and the controller deployments.
- Second commit to create the `clusters/staging/flux-system/gotk-sync.yaml` file
  which contains the Flux `GitRepository` and `Kustomization` custom resources for setting up the cluster reconciliation.

This Flux CLI will perform the following actions on the cluster:

- Creates a Kubernetes Secret named `flux-system` in the `flux-system` namespace that contains the bot PAT.
- Builds the `cluster/staging/flux-system` kustomize overlays with the multi-tenancy patches and
  applies the generated manifests to the cluster to kick off the reconciliation.

From this point on, the Flux controllers will reconcile the cluster state with the desired state, any changes
to the `cluster/staging` directory in the `d1-fleet` repository will be automatically applied to the cluster.

### Bootstrap with the enterprise version

When using the ControlPLane enterprise distribution for Flux, you need to create a 
Kubernetes Image Pull Secret for the enterprise registry in the `flux-system` namespace.

```shell
flux create secret oci flux-enterprise-auth \
  --url=ghcr.io \
  --username=flux \
  --password=$FLUX_ENTERPRISE_TOKEN
```

Then run the bootstrap command by specifying the enterprise registry and the image pull secret:

```shell
flux bootstrap github \
  --registry=ghcr.io/controlplaneio-fluxcd/disroless \
  --image-pull-secret=flux-enterprise-auth \
  --components-extra=image-reflector-controller,image-automation-controller \
  --owner=controlplaneio-fluxcd \
  --repository=d1-fleet \
  --branch=main \
  --token-auth \
  --path=clusters/staging
```

Another option is to copy the images from the ControlPLane registry to your organization's registry
and use the `--registry` flag to point to your registry.
