---
title: "Running Porch on GKE"
type: docs
weight: 2
description:
---

You can install Porch by either using one of the
[released versions](https://github.com/nephio-project/porch/releases), or building Porch from sources.

## Prerequisites

{{% alert title="Note" color="primary" %}}

Porch should run on any Kubernetes cluster and should work on any cloud. We have just started by documenting one
known-good configuration: GCP and GKE. We would welcome comparable installation instructions or feedback from people
that try it out on other clouds / configurations.

{{% /alert %}}

To run one of the [released versions](https://github.com/nephio-project/porch/releases) of Porch on GKE, you will
need:

* A [GCP Project](https://console.cloud.google.com/projectcreate)
* [gcloud](https://cloud.google.com/sdk/docs/install)
* [kubectl](https://kubernetes.io/docs/tasks/tools/); you can install it via `gcloud components install kubectl`
* [porchctl](https://github.com/nephio-project/porch/releases/download/dev/porchctl.tgz)
* Command line utilities such as *curl*, *tar*

To build and run Porch on GKE, you will also need:

* A container registry which will work with your GKE cluster.
  [Artifact Registry](https://console.cloud.google.com/artifacts) or
  [Container Registry](https://console.cloud.google.com/gcr) work well though you can use others too.
* [go 1.21](https://go.dev/dl/) or newer
* [docker](https://docs.docker.com/get-docker/)
* [Configured docker credential helper](https://cloud.google.com/sdk/gcloud/reference/auth/configure-docker)
* [git](https://git-scm.com/)
* [make](https://www.gnu.org/software/make/)

## Getting Started

Make sure your gcloud is configured with your project (alternatively, you can augment all following gcloud
commands below with --project flag):

```bash
gcloud config set project YOUR_GCP_PROJECT
```

Select a GKE cluster or create a new one:

```bash
gcloud services enable container.googleapis.com
gcloud container clusters create-auto --region us-central1 porch-dev
```
{{% alert title="Note" color="primary" %}}

For development of Porch, in particular for running Porch tests, Standard GKE cluster is currently preferable. Select a
[GCP region](https://cloud.google.com/compute/docs/regions-zones#available) that works best for your needs:

 ```bash
gcloud services enable container.googleapis.com
gcloud container clusters create --region us-central1 porch-dev
```

And ensure *kubectl* is targeting your GKE cluster:

```bash
gcloud container clusters get-credentials --region us-central1 porch-dev
```
{{% /alert %}}

## Run Released Version of Porch

To run a released version of Porch, download the release configuration bundle from
[Porch release page](https://github.com/nephio-project/porch/releases).

Untar and apply the *porch_blueprint.tar.gz* configuration bundle. This will install:

* Porch server
* [configsync](https://kpt.dev/gitops/configsync/)

```bash
mkdir porch-install
tar xzf ~/Downloads/porch_blueprint.tar.gz -C porch-install
kubectl apply -f porch-install
kubectl wait deployment --for=condition=Available porch-server -n porch-system
```

You can verify that Porch is running by querying the api-resources:

```bash
kubectl api-resources | grep porch
```
Expected output will include:

```bash
repositories                                   config.porch.kpt.dev/v1alpha1          true         Repository
packagerevisionresources                       porch.kpt.dev/v1alpha1                 true         PackageRevisionResources
packagerevisions                               porch.kpt.dev/v1alpha1                 true         PackageRevision
```

To install configsync:

```bash
echo "
apiVersion: configmanagement.gke.io/v1
kind: ConfigManagement
metadata:
  name: config-management
spec:
  enableMultiRepo: true
" | kubectl apply -f -
```

## Run Custom Build of Porch

To run custom build of Porch, you will need additional [prerequisites](#prerequisites). The commands below use
[Google Container Registry](https://console.cloud.google.com/gcr).

Clone this repository into *${GOPATH}/src/github.com/GoogleContainerTools/kpt*.

```bash
git clone https://github.com/GoogleContainerTools/kpt.git "${GOPATH}/src/github.com/GoogleContainerTools/kpt"
```

[Configure](https://cloud.google.com/sdk/gcloud/reference/auth/configure-docker) docker credential helper for your
repository.

If your use case doesn't require Porch to interact with GCP container registries, you can build and deploy Porch by
running the `gcr.io/YOUR-PROJECT-ID/porch-server:SHORT-COMMIT-SHA` command. It will build and push Porch Docker images into (by default) Google Container Registry
named (example shown is the Porch server image).

```bash
IMAGE_TAG=$(git rev-parse --short HEAD) make push-and-deploy-no-sa
```

If you want to use a different repository, you can set IMAGE_REPO variable
(see [Makefile](https://github.com/nephio-project/porch/blob/main/Makefile#L33) for details).

The `make push-and-deploy-no-sa` target will install Porch but not configsync. You can install configsync in your k8s
cluster manually following the
[documentation](https://github.com/GoogleContainerTools/kpt-config-sync/blob/main/docs/installation.md).

{{% alert title="Note" color="primary" %}}

The -no-sa (no service account) targets create Porch deployment
configuration which does not associate Kubernetes service accounts with GCP
service accounts. This is sufficient for Porch to integrate with Git repositories
using Basic Auth, for example GitHub.

As above, you can verify that Porch is running by querying the api-resources:

```bash
kubectl api-resources | grep porch
```
{{% /alert %}}

### Workload Identity

[Workload Identity](https://cloud.google.com/kubernetes-engine/docs/concepts/workload-identity) is a simple way to
access Google Cloud services from Porch.

#### Google Cloud Source Repositories

[Cloud Source Repositories](https://cloud.google.com/source-repositories) can be access using workload identity,
removing the need to store credentials in the cluster.

To set it up, create the necessary service accounts and give it the required roles:

```bash
GCP_PROJECT_ID=$(gcloud config get-value project)

# Create GCP service account (GSA) for Porch server.
gcloud iam service-accounts create porch-server

# We want to create and delete images. Assign IAM roles to allow repository
# administration.
gcloud projects add-iam-policy-binding ${GCP_PROJECT_ID} \
  --member "serviceAccount:porch-server@${GCP_PROJECT_ID}.iam.gserviceaccount.com" \
  --role "roles/source.admin"

gcloud iam service-accounts add-iam-policy-binding porch-server@${GCP_PROJECT_ID}.iam.gserviceaccount.com \
  --role roles/iam.workloadIdentityUser \
  --member "serviceAccount:${GCP_PROJECT_ID}.svc.id.goog[porch-system/porch-server]"

# We need to associate the Kubernetes Service Account (KSA)
# with the GSA by annotating the KSA.
kubectl annotate serviceaccount porch-server -n porch-system \
  iam.gke.io/gcp-service-account=porch-server@${GCP_PROJECT_ID}.iam.gserviceaccount.com
```

Build Porch, push images, and deploy Porch server and controllers using the `make` target that adds workload identity
service account annotations:

```bash
IMAGE_TAG=$(git rev-parse --short HEAD) make push-and-deploy
```

As above, you can verify that Porch is running by querying the api-resources:

```bash
kubectl api-resources | grep porch
```

To register a repository, use the following command:

```bash
porchctl repo register --repo-workload-identity --namespace=default https://source.developers.google.com/p/<project>/r/<repo>
```

#### OCI

To integrate with OCI repositories such as
[Artifact Registry](https://console.cloud.google.com/artifacts) or
[Container Registry](https://console.cloud.google.com/gcr), Porch relies on
[workload identity](https://cloud.google.com/kubernetes-engine/docs/how-to/workload-identity).

For that use case, create service accounts and assign roles:

```bash
GCP_PROJECT_ID=$(gcloud config get-value project)

# Create GCP service account for Porch server.
gcloud iam service-accounts create porch-server
# Create GCP service account for Porch sync controller.
gcloud iam service-accounts create porch-sync

# We want to create and delete images. Assign IAM roles to allow repository
# administration.
gcloud projects add-iam-policy-binding ${GCP_PROJECT_ID} \
  --member "serviceAccount:porch-server@${GCP_PROJECT_ID}.iam.gserviceaccount.com" \
  --role "roles/artifactregistry.repoAdmin"

gcloud iam service-accounts add-iam-policy-binding porch-server@${GCP_PROJECT_ID}.iam.gserviceaccount.com \
  --role roles/iam.workloadIdentityUser \
  --member "serviceAccount:${GCP_PROJECT_ID}.svc.id.goog[porch-system/porch-server]"

gcloud projects add-iam-policy-binding ${GCP_PROJECT_ID} \
  --member "serviceAccount:porch-sync@${GCP_PROJECT_ID}.iam.gserviceaccount.com" \
  --role "roles/artifactregistry.reader"

gcloud iam service-accounts add-iam-policy-binding porch-sync@${GCP_PROJECT_ID}.iam.gserviceaccount.com \
  --role roles/iam.workloadIdentityUser \
  --member "serviceAccount:${GCP_PROJECT_ID}.svc.id.goog[porch-system/porch-controllers]"
```
