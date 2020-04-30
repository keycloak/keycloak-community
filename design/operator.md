# Observerability

* **Status**: Notes
* **JIRA**: [KEYCLOAK-10036](https://issues.jboss.org/browse/KEYCLOAK-10036)


## PoC Video
https://www.youtube.com/watch?v=nGsARoHk_Dw

## Motivation

* Simplify user experience on Kubernetes/OpenShift
* Increase Keycloak availability though Operator HUB

## Related work

The Integreately Team have come up with a [working solution for Keycloak Operator](https://github.com/integr8ly/keycloak-operator).

## Introduction to the Operators

In this document, we consider the term Operator as Operator SDK and all the tooling
providing seamless experience for OpenShift 4. An Operator simplifies deployment and
management of all the services targeted for OpenShift platform.

An Operator follows the Reconsiler Pattern (described in the Cloud Native Infrastructure book).
It creates a target infrastructure state, gathers information about the current state and then reconsiles
based on it.

For more information, please take a look at the [Operator SDK documentation](https://github.com/operator-framework/operator-sdk).

## Operator capabilities

Here’s a short list and a brief description what Operator can do today:

### Keycloak provisioning

The Operator can provision (and deprovision) RHSSO instances based on Keycloak CRD. The Operator will also spin up a Postgresql database and will create a Secret with an Admin credentials. It is also possible to attach different RHSSO plugins (such as metrics SPI plugin) and finalizers (that are responsible for cleaning up the resources (routes, PVs etc) allocated during provisioning phase).

### Realm provisioning

The Operator is also capable of provisioning RHSSO Realms. The KeycloakRealm CRD is very similar to what RHSSO exported JSON has. One of the most important things is that KeycloakRealm CRD contains information about clients.

### Backups

It is also possible to use a Cron Job for backing up a RHSSO installation. The backup definition needs to be put into Keycloak CRD. Such definition contains an image responsible for backing up the database, encryption key and an AWS (S3) credentials secret name.

## Our plans

We plan to take over the existing implementation that the Integreately Team have done (thank you!). Further work can be divided into stages:

* [Stage 1:](https://issues.jboss.org/browse/KEYCLOAK-10318) Basic install capabilities:
  * Transition Keycloak Operator into Keycloak organization
  * Setup a simple testing with Travis
  * Setup automatic deployments with Travis (or Jenkins)
  * Tidy up the Readme file and implement basic documentation
  * Publish the Operator to the Operator HUB
  * Make sure everything work out of the box on Kubernetes
* [Stage 2:](https://issues.jboss.org/browse/KEYCLOAK-10319) Productization:
  * Productization of Stage 1 (in other words, release RHSSO with all this bits)

## Further work

Here's a list of the features we're considering for the future:

* Securing application automatically with Gatekeeper or Envoy JWT proxy
* Clients as CRDs
* Seamless upgrades

### Features mentioned (or requested) by the Community

* Support multiple Kubernetes clusters (multi-cluster HA support)
* Kerberos support (referencing a keytab file from a secret for example)