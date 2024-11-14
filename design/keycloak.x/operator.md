# Keycloak.X Operator

## Motivation for a new operator

The current Operator made in Go Lang served us and the community well so far, but increasing challenges are paving the road for a complete re-write.

* The codebase is hard to maintain because of organic growth and accumulated technical debt
* The Keycloak community is more keen to Java and there is less Go Lang expertise
* The current project needs some high-cost maintenance tasks to be performed in order to use the latest features (e.g. webhooks) and receive the latest fixes and patches, specifically:
    * upgrading Go lang version from 1.13 to 1.17
    * upgrading the Operator SDK and the dependencies

  Those upgrades will require creating a completely new project, using different libraries, moving, and in some cases, rewriting the components, e.g. the whole testsuite.
* The current approach around CRDs no longer fits the long term vision for cloud-native Keycloak as it is very error-prone and fragile.
* A Java operator can share business objects with the Keycloak main codebase increasing the code-reuse and dramatically reducing the chances of introducing bugs in the translation process from Kubernetes resources.
* A unified codebase will make it easy to test and debug the entire system.
* A new operator will embrace from the ground up the new Cloud Native capabilities of upcoming Keycloak releases such as the Quarkus distribution and Store.X, making those first-class citizen overall improving the user experience.


## Features

---
**NOTE**

The primary target of the operator is to make it easy to achieve production grade installations of Keycloak.X.

---

### Configuring Keycloak deployment

The operator will use a CRD representing the Keycloak installation. The CRD will expose the following configuration options:
* custom Keycloak image; default base image will be used if not specified
* Git configuration to fetch static business objects configuration (see below)
* manual horizontal and vertical scaling
* pod and node affinity, toleration
* SSL config, truststore

Since most of the configuration options will come from the Keycloak.X distribution itself, the CRD will also expose appropriate fields for passing any distribution related options to the container, like database connection details (obviously without any credentials), SPIs configuration, etc.

#### PodTemplate

The new operator will provide specific and opinionated CRD fields to tune the Keycloak deployment for the most common use-cases, but, one of those knobs will be a customizable [`PodTemplate`](https://kubernetes.io/docs/concepts/workloads/pods/#pod-templates) and the ability to add additional, arbitrary `Volumes`.
The provided `PodTemplate` will be merged with the properties set by the operator as an "escape hatch" to configure any Kubernetes property is not explicitly exposed by the CRD API.

This approach has been already proved to be successful by, among others, [Flink](https://nightlies.apache.org/flink/flink-docs-master/docs/deployment/resource-providers/native_kubernetes/#pod-template) and [Spark](https://spark.apache.org/docs/latest/running-on-kubernetes.html#pod-template) projects.

### Configuring Keycloak business objects using Kubernetes resources

The new operator will expose two main ways of configuring business objects in Keycloak (e.g.: Realms, Roles, Clients, etc.) in addition to the built-in Dynamic configuration through the REST API/UI console:
* Static file based configuration stored in Git to enable GitOps operations
* Static configuration through Kubernetes API

Static configuration will be strictly read-only, therefore two-way syncing is not going to be needed.

Static configuration is going to provide an immutable and cloud native way for managing business objects in Keycloak that can be easily moved between environments (e.g. dev, stage, prod) in a predictable manner. This feature will leverage the new Store.X which enables federation of the configuration from multiple sources (static/dynamic) by re-architecting the storage layer.

#### Static configuration through Git

The `Keycloak` CRD will enable defining a specific commit (identified by an hash) in a Git repository containing the static configuration in the form of JSON/YAML files. To update the configuration the user will simply change a commit hash in a `Keycloak` CR and the operator will roll out the new configuration to all pods.

#### Static configuration through Kubernetes API

The operator will leverage dedicated CRD(s), initially, there will be only one `Realm` CRD directly translated from Keycloak's [RealmRepresentation](https://github.com/keycloak/keycloak/blob/c7134fd5390d7c650b3dfd4bd2a2855157042271/core/src/main/java/org/keycloak/representations/idm/RealmRepresentation.java). A Realm includes all subresources. As a result, it is going to be possible to configure every object in Keycloak through this CR even though for some of them it won't be recommended (e.g. Users). To implement this, the operator will simply translate the CRs to YAML files and mount them to Keycloak pods, again leveraging Store.X.

It will be possible to store any credentials as Secrets in K8s leveraging [Keycloak Vault functionality](https://www.keycloak.org/docs/latest/server_admin/index.html#_vault-administration) where possible.

It's purpose of the upcoming Store.X initiative to provide a full-fledged static configuration backend for Keycloak but there will be a mid-term preview to enable bulk imports at startup time leveraging the REST api.

### Keycloak versions alignment

The operator will have its release cycle aligned with Keycloak. Each operator version will support only one corresponding Keycloak version.

### Upgrades

In order to upgrade to a newer Keycloak version, the operator will be upgraded first to ensure full compatibility with the operand.

If custom Keycloak image is not used, the operator will use a default base image. After the operator is upgraded, it automatically upgrades Keycloak too using a newer base image.

In case a custom Keycloak image is used, the image will need to be rebuilt to perform the upgrade. This is not going to be operator's responsibility as building a custom image often requires a complex process incl. CI/CD pipelines. After the operator is upgraded, it won't manage any existing Keycloak instances until its custom image is manually rebuilt using the right Keycloak base image aligned with the operator and updated and the image coordinates are updated in the CR.

Store.X will optionally allow zero-downtime rolling upgrades (a Keycloak upgrade performed pod by pod) that will ensure that Keycloak cluster will remain available even when upgrade fails on one of the pods.
"Recreate" upgrade strategy (all pods gracefully shut down and re-created with new image) will be also available.

### Reaugmentation process in Kubernetes

We will be leveraging Kubernetes volumes to act as "caches" for the [augmented/configured](https://quarkus.io/guides/reaugmentation) version of Keycloak.
An initial POC to show the concept has been drafted here:
https://github.com/andreaTP/poc-mutable-jar

We will use Kubernetes volumes to cache the augmented version of the binaries.
The artifacts in the kubernetes volume will be produced by an init-container and the operation is going to result in a noop in case the volume has already been populated by a compatible augmentation.

### Connecting to a database

A Postgres DB instance will have to be provisioned externally, it's not Keycloak Operator's responsibility to manage a database. The DB credentials will be fetched from K8s Secrets.

In long-term plan we'll add a limited integration with a Postgres Operator to leverage its backup functionalities for Keycloak upgrades.

### Observability

The operator will provide CR metrics as well as it will provide integration with Prometheus, Grafana and AlertManager for both operator and operand metrics. This will be addressed in an upcoming design proposal.

### Ingresses

The operator will provide an out-of-the-box experience using an opinionated default Ingress (Route on OpenShift) configuration. This configuration will support further "manual" modification. Additionally, it will be possible to completely disable this feature.

### Outgoing requests proxy settings

The operator will respect the standard `HTTP_PROXY`, `HTTPS_PROXY` and `NO_PROXY` environmental variables for any outgoing requests. These variables are by default [set by OLM](https://docs.openshift.com/container-platform/4.9/operators/admin/olm-configuring-proxy-support.html). The operator will also pass them to Keycloak pods to leverage the built-in support for them. This behaviour will be overridable. 


## Codebase

The code for the new operator will be organized as a Maven sub-module in the main GitHub `keycloak/keycloak` repository.
Dependency management will automatically piggy-back on the Keycloak BOM of the Quarkus distribution guaranteeing compliance of the used library versions.

It will use the Java Operator SDK and its Quarkus extension. This implies the usage of Fabric8 K8s client.

## Kubernetes compatibility matrix
* OpenShift >=4.7
* Vanilla Kubernetes >=1.20

Other Kubernetes distributions are supported only in the best effort mode.

## Distribution

The Operator deployment is going to be performed leveraging OLM providing both CLI approach via `Subscription` objects for managing the operator installation, and UI in OpenShift. The Operator as such is going to be distributed as a container image.

## Migration from the old operator
No direct migration path. Generic migration steps will be documented.

## Future considerations

### Autonomous operator

Our long-term vision includes making the operator autonomous to some extent, basically making it a [Level 5 operator](https://operatorframework.io/operator-capabilities/). It should be able to understand the operand's metrics and reflect them while automatically scaling and healing Keycloak deployment.