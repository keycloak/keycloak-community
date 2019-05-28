# Keycloak X - Distro

* **Status**: Notes
* **JIRA**: [KEYCLOAK-11322](https://issues.jboss.org/browse/KEYCLOAK-11322)


The aim is that as much as possible of the current codebase will be consumed both by the current Keycloak release and
the new Quarkus powered Keycloak.X.


## Distribution

We will soon provide a downloadable Keycloak.X distribution alongside regular Keycloak releases. We are also planning
to create a container image of course.


## Configuration

Introduce a single Keycloak configuration file. We are still discussing if this will be properties based, JSON or
YAML.

One key requirement is the ability to easily configure Keycloak.X in a container, which should be achievable through
exposing all config options as environment variables.


## Resources

* Check out [WIP](https://github.com/keycloak/keycloak/tree/master/quarkus) to try it out
* Join [#dev-keycloak-x](https://keycloak.zulipchat.com/#narrow/stream/208073-dev-keycloak.2Ex) on [Zulip Chat](https://keycloak.zulipchat.com) to participate