# Offline Session Lazy Loading

* **Status**: Draft #1
* **JIRA**: [KEYCLOAK-11019](https://issues.redhat.com/browse/KEYCLOAK-11019)

## Motivation

To make it fast to startup Keycloak with a large number of Offline Sessions.

Keycloak has become to be used with a large number of offline sessions, around 10M. In such cases, the startup time tends to be too long to be acceptable because Keycloak preloads all offline sessions in the startup. Of course, we can shorten the time by tuning some factors like `sessionsPerSegment`, Keycloak server's spec, DB server's spec, and so on. However, there is a limit to the startup time we can shorten by tuning. In one example,  with 6M offline sessions, the limit to startup time was 50 minutes.

We'd like to break through the limit to startup time. To break through the limit, we need to consider resolving the root cause, that is, consider changing the offline session loading method from preloading all offline sessions in the startup.

To realize this, here we'd like to discuss "Offline Session Lazy Loading".

## Use-cases

Currently, there are several use cases we need to stop the Keycloak server (when clustering, stop all Keycloak servers). In such use cases, we also need to restart the servers, and we need to wait a long startup time.

### 1. Version-up

Not only when major version-up which there is a possibility to be changed the DB schema, but also when minor version-up or micro version-up, stopping all Keycloak servers are currently recommended ([Support for RH-SSO upgrade with zero downtime (rolling upgrades)](https://access.redhat.com/solutions/3742021)).
[Zero downtime upgrade](https://issues.redhat.com/browse/KEYCLOAK-7301) is planned, but it supposed to be difficult if the DB schema is changed.

### 2. Configuration changes

When we change the configuration (like standalone.xml), we also are recommended to stop all Keycloak servers, because it's not ensured to work well with configuring several Keycloak servers with different configurations.



## How the feature will be used

Users can be select the loading method (preload or lazyload) in the configuration file (like standalone.xml).



## Implementation details

We must list up when offline sessions are accessed, and how (for example, get only one offline session, update several offline sessions, get the number of offline sessions, or so on) offline sessions are accessed.

And we also have to decide on more detailed parts. For example, if get only one offline session, load only one offline session or load several offline sessions in some range.

TBD

