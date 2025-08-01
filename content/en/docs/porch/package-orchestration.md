---
title: "Package Orchestration"
type: docs
weight: 2
description: 
---

## Why

People who want to take advantage of the benefits of [Configuration as Data](config-as-data.md) can do so today using
a [kpt](https://kpt.dev) CLI and the kpt function ecosystem, including its [functions catalog](https://catalog.kpt.dev/).
Package authoring is possible using a variety of editors with [YAML](https://yaml.org/) support. That said, a delightful
UI experience of WYSIWYG package authoring which supports broader package lifecycle, including package authoring with
*guardrails*, approval workflow, package deployment, and more, is not yet available.

Porch *Package Orchestration* (Porch) is part of the Nephio implementation of a Configuration as Data approach. It offers an API and
a CLI that enables building that delightful UI experience for supporting the configuration lifecycle.

## Core Concepts

This section briefly describes core concepts of package orchestration:

***Package***: Package is a collection of related configuration files containing configuration of [KRM][krm]
**resources**. Specifically, configuration packages are [kpt packages](https://kpt.dev/).

***Repository***: Repositories store packages. For example [git][] or [OCI][oci]. ([more details](#repositories))

Packages are sequentially ***versioned***; multiple versions of the same package may exist in a repository.
([more details](#package-versioning))

A package may have a link (URL) to an ***upstream package*** (a specific version) from which it was cloned.
([more details](#package-relationships))

Package may be in one of several lifecycle stages:

* ***Draft*** - package is being created or edited. The package contents can be modified but package is not ready to be
  used (i.e. deployed)
* ***Proposed*** - author of the package proposed that the package be published
* ***Published*** - the changes to the package have been approved and the package is ready to be used. Published
  packages can be deployed or cloned

***Functions*** (specifically, [KRM functions][krm functions]) can be applied to packages to mutate or validate resources
within them. Functions can be applied to a package to create specific package mutation while editing a package draft,
functions can be added to package's Kptfile [pipeline][].

A repository can be designated as ***deployment repository***. *Published* packages in a deployment repository are
considered deployment-ready. ([more details](#deployment))

## Core Components of Configuration as Data Implementation

The Core implementation of Configuration as Data, *CaD Core*, is a set of components and APIs which collectively enable:

* Registration of repositories (Git, OCI) containing kpt packages and the discovery of packages
* Management of package lifecycles, including authoring, versioning, deletion, creation and mutations of a package draft,
  process of proposing the package draft, and publishing of the approved package
* Package lifecycle operations such as:

  * assisted or automated rollout of package upgrade when a new version of the upstream package version becomes
    available (3 way merge)
  * rollback of a package to previous version
* Deployment of packages from deployment repositories and observability of their deployment status
* Permission model that allows role-based access control

### High-Level Architecture

At the high level, the Core CaD functionality comprises:

* a generic (i.e. not task-specific) package orchestration service implementing

  * package repository management
  * package discovery, authoring and lifecycle management

* [porchctl](user-guides/porchctl-cli-guide.md) - a Git-native, schema-aware, extensible client-side tool for managing KRM packages
* a GitOps-based deployment mechanism (for example [configsync][]), which distributes and deploys configuration, and
  provides observability of the status of deployed resources
* a task-specific UI supporting repository management, package discovery, authoring, and lifecycle

![CaD Core Architecture](/static/images/porch/CaD-Core-Architecture.svg)

## CaD Concepts Elaborated

Concepts briefly introduced above are elaborated in more detail in this section.

### Repositories

Porch and [configsync][] currently integrate with [git][] repositories, and there is an existing design to add OCI
support to kpt. Initially, the Package Orchestration service will prioritize integration with [git][], and support for
additional repository types may be added in the future as required.

Requirements applicable to all repositories include: ability to store packages, their versions, and sufficient metadata
associated with package to capture:

* package dependency relationships (upstream - downstream)
* package lifecycle state (draft, proposed, published)
* package purpose (base package)
* (optionally) customer-defined attributes

At repository registration, customers must be able to specify details needed to store packages in appropriate locations
in the repository. For example, registration of a Git repository must accept a branch and a directory.

{{% alert title="Note" color="primary" %}}

A user role with sufficient permissions can register a package or function repository, including repositories
containing functions authored by the customer, or other providers. Since the functions in the registered repositories
become discoverable, customers must be aware of the implications of registering function repositories and trust the
contents thereof.

{{% /alert %}}

### Package Versioning

Packages are sequentially versioned. The important requirements are:

* ability to compare any 2 versions of a package to be either "newer than", equal, or "older than" relationship
* ability to support automatic assignment of versions
* ability to support [optimistic concurrency][optimistic-concurrency] of package changes via version numbers
* a simple model which easily supports automation

We use a simple integer sequence to represent package versions.

### Package Relationships

Kpt packages support the concept of ***upstream***. When a package is cloned from another, the new package
(called the ***downstream*** package) maintains an upstream link to the specific version of the package from which it was
cloned. If a new version of the upstream package becomes available, the upstream link can be used to update the downstream package.

### Deployment

The deployment mechanism is responsible for deploying configuration packages from a repository and affecting the live
state. Because the configuration is stored in standard repositories (Git, and in the future OCI), the deployment
component is pluggable. By default, [configsync][] is the deployment mechanism used by CaD Core implementation but
others can be used as well.

Here we highlight some key attributes of the deployment mechanism and its integration within the CaD Core:

* _Published_ packages in a deployment repository are considered ready to be deployed
* configsync supports deploying individual packages and whole repositories. For Git specifically that translates to a
  requirement to be able to specify repository, branch/tag/ref, and directory when instructing configsync to deploy a
  package.
* _Draft_ packages need to be identified in such a way that configsync can easily avoid deploying them.
* configsync needs to be able to pin to specific versions of deployable packages in order to orchestrate rollouts and
  rollbacks. This means it must be possible to GET a specific version of a package.
* configsync needs to be able to discover when new versions are available for deployment.

## Package Orchestration - Porch

Having established the context of the CaD Core components and the overall architecture, the remainder of the document
will focus on **Porch** - Package Orchestration service.

To reiterate the role of Package Orchestration service among the CaD Core components, it is:

* [Repository Management](#repository-management)
* [Package Discovery](#package-discovery)
* [Package Authoring](#package-authoring) and Lifecycle

In the following section we'll expand more on each of these areas. The term _client_ used in these sections can be
either a person interacting with the UI such as a web application or a command-line tool, or an automated agent or
process.

### Repository Management

The repository management functionality of Package Orchestration service enables the client to:

* register, unregister, update registration of repositories, and discover registered repositories. Git repository
  integration will be available first, with OCI and possibly more delivered in the subsequent releases.
* manage repository-wide upstream/downstream relationships, i.e. designate default upstream repository from which
  packages will be cloned.
* annotate repository with metadata such as whether repository contains deployment ready packages or not; metadata can
  be application or customer specific

### Package Discovery

The package discovery functionality of Package Orchestration service enables the client to:

* browse packages in a repository
* discover configuration packages in registered repositories and sort/filter based on the repository containing the
  package, package metadata, version, package lifecycle stage (draft, proposed, published)
* retrieve resources and metadata of an individual package, including latest version or any specific version or draft
  of a package, for the purpose of introspection of a single package or for comparison of contents of multiple
  versions of a package, or related packages
* enumerate _upstream_ packages available for creating (cloning) a _downstream_ package
* identify downstream packages that need to be upgraded after a change is made to an upstream package
* identify all deployment-ready packages in a deployment repository that are ready to be synced to a deployment target
  by configsync
* identify new versions of packages in a deployment repository that can be rolled out to a deployment target by configsync

### Package Authoring

The package authoring and lifecycle functionality of the package Orchestration service enables the client to:

* Create a package _draft_ via one of the following means:

  * an empty draft 'from scratch' (`porchctl rpkg init`)
  * clone of an upstream package (`porchctl rpkg clone`) from either a
    registered upstream repository or from another accessible, unregistered, repository
  * edit an existing package (`porchctl rpkg pull`)
  * roll back / restore a package to any of its previous versions
    (`porchctl rpkg pull` of a previous version)

* Push changes to a package _draft_. In general, mutations include adding/modifying/deleting any part of the package's
  contents. Some specific examples include:

  * add/change/delete package metadata (i.e. some properties in the `Kptfile`)
  * add/change/delete resources in the package
  * add function mutators/validators to the package's pipeline
  * add/change/delete sub-package
  * retrieve the contents of the package for arbitrary client-side mutations (`porchctl rpkg pull`)
  * update/replace the package contents with new contents, for example results of a client-side mutations by a UI
    (`porchctl rpkg push`)

* Rebase a package onto another upstream base package or onto a newer version of the same package (to
  aid with conflict resolution during the process of publishing a draft package)

* Get feedback during package authoring, and assistance in recovery from merge conflicts, invalid package changes, guardrail violations

* Propose for a _draft_ package be _published_.
* Apply an arbitrary decision criteria, and by a manual or automated action, approve (or reject) proposal of a _draft_
  package to be _published_.
* Perform bulk operations such as:

  * Assisted/automated update (upgrade, rollback) of groups of packages matching specific criteria (i.e. base package
    has new version or specific base package version has a vulnerability and should be rolled back)
  * Proposed change validation (pre-validating change that adds a validator function to a base package)

* Delete an existing package.

#### Authoring & Latency

An important goal of the Package Orchestration service is to support building of task-specific UIs. In order to deliver
low latency user experience acceptable to UI interactions, the innermost authoring loop (depicted below) will require:

* high performance access to the package store (load/save package) with caching
* low latency execution of mutations and transformations on the package contents
* low latency [KRM function][krm functions] evaluation and package rendering (evaluation of package's function
  pipelines)

![Inner Loop](/static/images/porch/Porch-Inner-Loop.svg)

#### Authoring & Access Control

A client can assign actors (persons, service accounts) to roles that determine which operations they are allowed to
perform in order to satisfy requirements of the basic roles. For example, only permitted roles can:

* manipulate repository registration, enforcement of repository-wide invariants and guardrails
* create a draft of a package and propose the draft be published
* approve (or reject) the proposal to publish a draft package
* clone a package from a specific upstream repository
* perform bulk operations such as rollout upgrade of downstream packages, including rollouts across multiple downstream
  repositories
* etc.

### Porch Architecture

The Package Orchestration service, **Porch** is designed to be hosted in a [Kubernetes](https://kubernetes.io/) cluster.

The overall architecture is shown below, and includes also existing components (k8s apiserver and configsync).

![Porch Architecture](/static/images/porch/Porch-Architecture.svg)

In addition to satisfying requirements highlighted above, the focus of the architecture was to:

* establish clear components and interfaces
* support a low-latency package authoring experience required by the UIs

The Porch components are:

#### Porch Server

The Porch server is implemented as [Kubernetes extension API server][apiserver]. The benefits of using Kubernetes
extension API server are:

* well-defined and familiar API style
* availability of generated clients
* integration with existing Kubernetes ecosystem and tools such as `kubectl` CLI,
  [RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
* avoids requirement to open another network port to access a separate endpoint running inside k8s cluster (this is a
  distinct advantage over GRPC which we considered as an alternative approach)

Resources implemented by Porch include:

* `PackageRevision` - represents the _metadata_ of the configuration package revision stored in a _package_ repository.
* `PackageRevisionResources` - represents the _contents_ of the package revision

Note that each configuration package revision is represented by a _pair_ of resources which each present a different
view (or [representation][] of the same underlying package revision.

Repository registration is supported by a `Repository` [custom resource][crds].

**Porch server** itself comprises several key components, including:

* The *Porch aggregated apiserver* which implements the integration into the main Kubernetes apiserver, and directly
  serves API requests for the `PackageRevision`, `PackageRevisionResources` resources.
* Package orchestration *engine* which implements the package lifecycle operations, and package mutation workflows
* *CaD Library* which implements specific package manipulation algorithms such as package rendering (evaluation of
  package's function *pipeline*), initialization of a new package, etc. The CaD Library is shared with `kpt`
  where it likewise provides the core package manipulation algorithms.
* *Package cache* which enables both local caching, as well as abstract manipulation of packages and their contents
  irrespectively of the underlying storage mechanism (Git, or OCI)
* *Repository adapters* for Git and OCI which implement the specific logic of interacting with those types of package
  repositories.
* *Function runtime* which implements support for evaluating [kpt functions][functions] and multi-tier cache of
  functions to support low latency function evaluation

#### Function Runner

**Function runner** is a separate service responsible for evaluating [kpt functions][functions]. Function runner exposes
a [GRPC](https://grpc.io/) endpoint which enables evaluating a kpt function on the provided configuration package.

The GRPC technology was chosen for the function runner service because the [requirements](#grpc-api) that informed
choice of KRM API for the Package Orchestration service do not apply. The function runner is an internal microservice,
an implementation detail not exposed to external callers. This makes GRPC perfectly suitable.

The function runner also maintains a cache of functions to support low latency function evaluation. It achieves this through
two mechanisms available to it for evaluation of a function

**Executable Evaluation** approach executes the function within the Pod runtime through shell based invocation of function 
binary; for which function binaries are bundled inside the function runner image itself

**Pod Evaluation** approach is utilized when invoked function is not available via Executable Evaluation approach wherein
function runner pod starts the function pod corresponding to invoked function along with a front-end service. Once 
the pod and service are up and running, it's exposed GRPC endpoint is invoked for function evaluation, passing the input 
package. For this mechanism, function runner reads the list of functions and their images supplied via a config
file at startup, and spawns function pods, along with a corresponding front-end service for each configured function.
These function pods/services are terminated after a pre-configured period of inactivity (default 30 minutes) by function
runner and recreated on the next invocation.

#### CaD Library

The [kpt](https://kpt.dev/) CLI already implements foundational package manipulation algorithms in order to provide the
command line user experience, including:

* [kpt pkg init](https://kpt.dev/reference/cli/pkg/init/) - create an empty, valid, KRM package
* [kpt pkg get](https://kpt.dev/reference/cli/pkg/get/) - create a downstream package by cloning an upstream package;
  set up the upstream reference of the downstream package
* [kpt pkg update](https://kpt.dev/reference/cli/pkg/update/) - update the downstream package with changes from new
  version of upstream, 3-way merge
* [kpt fn eval](https://kpt.dev/reference/cli/fn/eval/) - evaluate a kpt function on a package
* [kpt fn render](https://kpt.dev/reference/cli/fn/render/) - render the package by executing the function pipeline of
  the package and its nested packages
* [kpt fn source](https://kpt.dev/reference/cli/fn/source/) and [kpt fn sink](https://kpt.dev/reference/cli/fn/sink/) -
  read package from local disk as a `ResourceList` and write package represented as `ResourcesList` into local disk

The same set of primitives form the foundational building blocks of the package orchestration service. Further, the
package orchestration service combines these primitives into higher-level operations (for example, package orchestrator
renders packages automatically on changes, future versions will support bulk operations such as upgrade of multiple
packages, etc).

The implementation of the package manipulation primitives in kpt was refactored (with initial refactoring completed, and
more to be performed as needed) in order to:

* create a reusable CaD library, usable by both kpt CLI and Package Orchestration service
* create abstractions for dependencies which differ between CLI and Porch, most notable are dependency on Docker for
  function evaluation, and dependency on the local file system for package rendering.

Over time, the CaD Library will provide the package manipulation primitives:

* create a valid empty package (init)
* update package upstream pointers (get)
* perform 3-way merge (update)
* render - core package rendering algorithm using a pluggable function evaluator to support:

  * function evaluation via Docker (used by kpt CLI)
  * function evaluation via an RPC to a service or appropriate function sandbox
  * high-performance evaluation of trusted, built-in, functions without sandbox

* heal configuration (restore comments after lossy transformation)

and both kpt CLI and Porch will consume the library. This approach will allow leveraging the investment already made
into the high quality package manipulation primitives, and enable functional parity between Kpt CLI and Package
Orchestration service.

## User Guide

Find the Porch User Guide in a dedicated
[document](https://github.com/kptdev/kpt/blob/main/site/guides/porch-user-guide.md).

## Open Issues/Questions

### Deployment Rollouts & Orchestration

__Not Yet Resolved__

Cross-cluster rollouts and orchestration of deployment activity. For example, package deployed by configsync in cluster
A, and only on success, the same (or a different) package deployed by configsync in cluster B.

## Alternatives Considered

### GRPC API

We considered the use of [GRPC]() for the Porch API. The primary advantages of implementing Porch as an extension
Kubernetes apiserver are:

* customers won't have to open another port to their Kubernetes cluster and can reuse their existing infrastructure
* customers can likewise reuse existing, familiar, Kubernetes tooling ecosystem

<!-- Reference links -->
[krm]: https://github.com/kubernetes/design-proposals-archive/blob/main/architecture/resource-management.md
[functions]: https://kpt.dev/book/02-concepts/03-functions
[krm functions]: https://github.com/kubernetes-sigs/kustomize/blob/master/cmd/config/docs/api-conventions/functions-spec.md
[pipeline]: https://kpt.dev/book/04-using-functions/01-declarative-function-execution
[Config Sync]: https://cloud.google.com/anthos-config-management/docs/config-sync-overview
[kpt]: https://kpt.dev/
[git]: https://git-scm.org/
[optimistic-concurrency]: https://en.wikipedia.org/wiki/Optimistic_concurrency_control
[apiserver]: https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/apiserver-aggregation/
[representation]: https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md#differing-representations
[crds]: https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/
[oci]: https://github.com/opencontainers/image-spec/blob/main/spec.md
