# Keycloak.X

* **Status**: Notes
* **JIRA**: [KEYCLOAK-10213](https://issues.jboss.org/browse/KEYCLOAK-10213)

## What are we trying to improve?

The first stable release of Keycloak was way back in 2014. As always when building software there are things that could
have been done better.

With Keycloak.X we are aiming to introduce some bigger changes to make Keycloak leaner, easier and more future-proof.

A few goals with Keycloak.X are:

* Make it easier to configure
* Make it easier to scale, including multi-site support
* Make it easier to extend
* Reduce startup time and memory footprint
* Support zero-downtime upgrades
* Support continuous delivery 

This work will be broken into several parts:

* A new and improved storage layer
* A new distribution powered by [Quarkus](https://quarkus.io)
* A new approach to custom providers

### [Distribution](distro.md)

Building a new distribution powered by Quarkus will allow us to significantly reduce startup time and memory footprint.

We will be able to create a leaner distribution in terms of size and dependencies as well. Reducing dependencies will
further reduce the number of CVEs in third-party libraries.

We are also planning to introduce a proper Keycloak configuration file, where we will document directly how to configure
everything related to Keycloak. In the current WildFly based distribution the configuration file is very complex as
it contains everything to configure the underlying application server, and more often than not it is required to refer
to WildFly documentation to figure out how to configure things properly.

### [Storage](storage.md)

The current storage layer is complex, especially when deployed to multiple-sites. It has a number of scalability issues
like the number of realms and clients. Sessions are only kept in-memory, which can be good for performance, but not
so great for scaling when you consider a large portion of sessions are idle and unused most of the time.

Exactly what the new storage layer will look like is still to be decided, but we know for sure that we want to:

* Reduce complexity with regards to configuring, SPIs and schema
* Support zero downtime upgrades
* Make sure we can scale to large number of realms and clients
* Make sure we can scale to millions of sessions, including support for persisting and passivation

### [Providers](providers.md)

Providers today have some issues that we would like to address. Including:

* Deprecation and versioned approach to SPIs - breaking changes to APIs are horrible in a continuous delivery world
* Polyglot - not everyone is a JavaEE developer, let's embrace that and allow more options when it comes to extending Keycloak
* Sand-boxing - allow safe customizations in a SaaS world 

### [Continuous Delivery](cd.md)

We are aiming to make it easier to use Keycloak in a continuous delivery world. This should consider Keycloak upgrades,
custom providers as well as configuration.

Keycloak upgrades should be seamless and there should not be any breaking changes, rather deprecation periods.

It should be possible to more easily manage and reproduce the config of Keycloak, including realm config, in different
environments. A developer should be able to try some config changes in a dev environment, push to a test environment, 
before finally making the changes live in a production environment.

### Contributing

We would love help from the community on Keycloak.X. You can contribute with code, with discussions or simply just trying
it out and giving us feedback. 

## Migration to Keycloak X

There will be a migration required to Keycloak.X. In fact there will be multiple migrations required as everything 
mentioned earlier will not be ready in one go.

It is an aim to make this migration as simple and painless as possible though.

## Timing

We are staring with the Quarkus powered distribution. The aim is to have a fully functional stable distribution by the
end of 2019, but we already have [a prototype](https://github.com/keycloak/keycloak/tree/master/quarkus) you can try out 
and contribute to.

In 2020 we are aiming to work on both the storage layer and providers. Hopefully, by the end of 2020 we will have most
if not everything sorted out.

We will continue to support the current Keycloak version in parallel with Keycloak.X and will give everyone plenty of
time to do the migration before we eventually will pull the plug on the old.