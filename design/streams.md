# Complement methods that operate on collections in the providers in `KeycloakSession` with `Stream`

* **Status**: Draft #1
* **JIRA**: [KEYCLOAK-14011](https://issues.redhat.com/browse/KEYCLOAK-14011)

## Motivation

Currently the providers (e.g. [RealmProvider](https://github.com/keycloak/keycloak/blob/9.0.3/server-spi/src/main/java/org/keycloak/models/RealmProvider.java), [ClientProvider](https://github.com/keycloak/keycloak/blob/9.0.3/server-spi/src/main/java/org/keycloak/models/ClientProvider.java)) accessible (in)directly from `KeycloakSession` operate on `List` and similar collections.

Consequently, the `List` operations are realized in memory even when only needed temporarily to be later filtered out like [in this piece of code](https://github.com/keycloak/keycloak/blob/9.0.3/services/src/main/java/org/keycloak/services/resources/admin/RealmAdminResource.java#L1198-L1202).

Thus the stream variants of the methods should be created, with backward compatibility in mind. This means that to each method with a list signature, e.g.:

```java
    List<ClientModel> getClients(RealmModel realm);
```

The change will be as follows:

```java
    @Deprecated
    default List<ClientModel> getClients(RealmModel realm) { /* use getClientsStream */ }

    default Stream<ClientModel> getClientsStream(RealmModel realm) {
        return getClients(realm).stream();
    }
```

This way, existing implementations will be able to still work, but the message that stream variant is preferred would be stated via `@Deprecated` annotation.

## Acceptance criteria

* All interfaces mentioned above have `Stream` counterparts for operations that currently operate on collections.
* REST endpoints preferentially use `Stream<T>` over the current collections.
* JPA preferentially use streams for obtaining query results [where applicable](https://thoughts-on-java.org/jpa-2-2s-new-stream-method-and-how-you-should-not-use-it/).



## Milestones

### M1 Proof of concept on client resources

Ensure that acceptance criteria are met on [ClientProvider](https://github.com/keycloak/keycloak/blob/9.0.3/server-spi/src/main/java/org/keycloak/models/ClientProvider.java).

### M2 Mark non-stream variants as `@Deprecated`

### M2 Update the rest of the resources

1. Ensure that acceptance criteria are met on [RealmProvider](https://github.com/keycloak/keycloak/blob/9.0.3/server-spi/src/main/java/org/keycloak/models/RealmProvider.java).
1. Ensure that acceptance criteria are met on [UserProvider](https://github.com/keycloak/keycloak/blob/9.0.3/server-spi/src/main/java/org/keycloak/models/UserProvider.java).
1. Ensure that acceptance criteria are met on [UserSessionProvider](https://github.com/keycloak/keycloak/blob/9.0.3/server-spi/src/main/java/org/keycloak/models/UserSessionProvider.java).
