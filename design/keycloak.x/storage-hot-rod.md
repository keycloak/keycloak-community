# Keycloak X - Storage - Infinispan/Hot Rod persistence layer

* **Status**: Draft
* **JIRA**: https://issues.redhat.com/browse/KEYCLOAK-17633
# Goals of this document

This document aims to address the following items:

* Follow up on no-downtime strategy presented in the [the Storage / Persistence
  layer proposal](storage-persistence.md)
* Present a way how to store and query data using Hot Rod Client and Remote
  Infinispan server so that we achieve the no-downtime requirements

# Non-Goals of this document

This document aims to _not_ address the following items:

* Provide a way how to replace the existing Infinispan layer
* Discuss the design of caching or its impact on performance

# Technologies

- **[Infinispan](https://infinispan.org)** - Infinispan is a distributed
  in-memory key/value data store with optional schema. It can be used both as an
  embedded Java library and as a language-independent service accessed remotely
  over a variety of protocols. In this proposal, we target Infinispan in the
  remote (client/server) scenario.
- **[Hot Rod](https://infinispan.org/docs/stable/titles/hotrod_java/hotrod_java.html)** - 
  Hot Rod is a wire protocol that Infinispan clients use to talk to a remote
  grid. It is a binary, platform-independent protocol that was developed in the
  open as a part of Infinispan.
- **[Protocol buffers](https://developers.google.com/protocol-buffers/)** -
  Protocol Buffers (Protobuf) is a lightweight binary media type for structured
  data. In our case, it provides interoperability between the client (Keycloak-X
  store) and the Infinispan server.

# Introduction

[The Storage / Persistence layer proposal](storage-persistence.md) suggests the
following strategy to achieve 0-downtime upgrade store:

1. The store can read objects from the oldest ones to the current version
2. The store can read objects from one version following the current one
3. Objects are only updated when written to.
4. Number of schema changes should be kept at absolute minimum
5. Schema changes can be postponed and run at chosen time (even if that means
   running a degraded service)

A simple implementation of the following design can be found in this [pull
request](https://github.com/keycloak/keycloak-playground/pull/11) into the
keycloak-playground repository. This repository is a playground project that
simplifies implementing and testing Keycloak-X-like no-downtime storage
implementations.

# Data structure definition

For describing a structure of data that is stored in the Infinispan server there
are Protobuf message definitions. Message definitions represents storage
representation of objects. Based on the message definition the Hot Rod client
knows how to marshal and unmarshal the data that is sent/received to/from
server. To make the Infinispan server able to query stored data, it is also
necessary to share message definitions with the Infinispan server.

An example of a message definition is following:

```protobuf
package nodowntimeupgrade;

message InfinispanObjectEntity {
    required int32 entitySchemaVersion = 1;
    required string id = 2;
    optional string name = 3;
    optional string clientTemplateId = 4;
}
```

This protobuf definition can be automatically generated from an annotated Java
class definition. For setting Protobuf message-specific settings there are
`@ProtoField` and `@ProtoDoc` (described in [Speeding up queries with
indices](#speeding-up-queries-with-indices) section) annotations.

```java
public class InfinispanObjectEntity {

    @ProtoField(number = 1, required = true)
    public int entitySchemaVersion = ModelVersion.VERSION_1.getVersion();

    @ProtoField(number = 2, required = true)
    public String id;

    @ProtoField(number = 3)
    @ProtoDoc("@Field(index = Index.YES, store = Store.YES, analyze = Analyze.YES, analyzer = @Analyzer(definition = \"keyword\"))")
    public String name;

    @ProtoField(number = 4)
    public String clientTemplateId;

}
```

Each field has its number. These numbers serve for identification during the
marshaling/unmarshaling process.

## Changes in object representations

### Addition of new fields

The advantage of Protobuf definitions is that they are flexible. It is possible
to add any number of fields on updates. For example, when we want to introduce a
new field named `clientScopeId`, we can do it by adding

```java
@ProtoField(number = 5)
public String clientScopeId;
```

to the message definition and updating the definition in the Infinispan server
and in the current `SerializationContext`. Then everything should work as
before. It is also possible to read objects with new fields from the old storage
version. The new field `clientScopeId` is skipped during unmarshaling.

### Changing field name/type

It is possible to change field names or types. However, such changes can
introduce backward incompatibility that we need to avoid due to no-downtime
requirements. Also, removing or renaming fields can break Ickle query searches,
see the [Querying objects](#querying-objects) section for more details. In this
section we show how to avoid backward incompatibilities on simple scenario
without considering migration for Ickle search queries.  

The steps will be illustrated on the following example: let us have a message
definition containing field `clientTemplateId` in the [_entity schema version_](storage-persistence.md#_def_entity_schema_version) 
`1` (`entitySchemaVersion = 1`). We want to replace it with `clientScopeId`. The
migration path is prescribed as `clientScopeId = "template-" +
clientTemplateId`).

For the example above, the steps to change field `clientTemplateId` to
`clientScopeId` is as follows:

1. We update the _entity schema_ to version `2` (`entitySchemaVersion = 2`) by
   adding `clientScopeId` to the Protobuf message definition so that it contains
   both `clientTemplateId` and `clientScopeId`
2. The [logical representation](storage-persistence.md#_def_logical_representation_) 
   only contains the new field (`clientScopeId`). The
   deprecated `clientTemplateId` field is taken into account in three cases:
   1. When writing to the storage and `clientScopeId` value starts with
      `"template-"` prefix. In such case, `clientTemplateId` is derived from
      `clientScopeId` by removing `"template-"` prefix and written together with
      `clientScopeId`. This is to support the reading of [storage representation](storage-persistence.md#_def_storage_representation_)
      with _entity schema version_ `2` older storage implementation. 
   2. When reading from the storage and the object has lower _entity schema
      version_. The `clientTemplateId` field is read from the storage, and
      migrated to `clientScopeId`. This step is necessary to read objects with
      _entity schema version_ 1 by the newer storage implementations.
   3. When querying objects. This part is discussed in the section [Querying
      objects](#querying-objects).
3. Then, there can be a new storage version `>= 3` which removes the deprecated
   `clientTemplateId`. Migrating to that storage version requires that all
   stored objects are migrated to _entity schema version_ `>= 2`. We will get
   back to the removal of fields later as it gets more complicated with regards
   to querying.

# Reading older storage representations

Each storage representation contains the `entitySchemaVersion` field that
represents its _entity schema version_. To fulfill the requirements for
0-downtime upgrades, we need to be able to read all older storage
representations of objects. 

As described in the previous section, unmarshaling of stored objects is flexible
enough to be able to read an older version when message definition contains all
fields in unmarshalled stream. Beware of premature removal of field definitions:
Unknown fields are ignored so even when their existence is transparent to the
reader, the information stored there would be lost.

In some cases, like the migration described in the previous section, we need to
migrate one field to another. In this case, Keycloak storage reads the storage
representation; checks what _entity schema version_ the read object has and
migrate it when necessary. The migration is done in the Keycloak storage code
after reading an older storage representation. The logical representation
contains the migrated field. For example, this is how the migration could look
like in the case of the previous migration path:

```java
public class EntityMigrationToVersion2 extends ObjectEntityDelegate {

    public EntityMigrationToVersion2(InfinispanObjectEntity delegate) {
        super(delegate);
    }

    @Override
    public String getClientScopeId() {
        if (super.getClientScopeId() == null && super.getClientTemplateId() != null) {
            return "template-" + super.getClientTemplateId();
        }

        return super.getClientScopeId();
    }
}
```

* When Keycloak storage corresponding to _entity schema version_ `2` detects
  that obtained [physical representation](storage-persistence.md#_def_physical_representation_) has an
  older version (in this case `1`) than the current (`2`), it wraps the physical
  representation into a migration object. 
* This migration object overrides getter methods for fields that require
  migration. In the example above, it overrides the method `getClientScopeId` so
  that the `clientTemplateId` field is used when `clientScopeId` is not set.
* The name for the migration object can be chosen arbitrarily.
  `EntityMigrationToVersion2` is just an example.
* The logical representation contains only `clientScopeId`; the presence of the
  `clientTemplateId` field is hidden in physical representation.
* Too long chain of delegations can cause performance overhead. It can be solved
  by the migration of storage representations to a newer version (in the future
  might be solved by some scheduled tasks functionality)

# Querying objects

Due to the fact that schema is known also on the Infinispan side, it is possible
to query objects based on field names. It is also possible to create indices to
speed up queries, more on that later in section [Speeding up queries with
indices](#speeding-up-queries-with-indices). Infinispan supports queries using
[Ickle query language](https://infinispan.org/docs/stable/titles/query/query.html#ickle-query-language).
Here is an example, how the Ickle query looks like:

```java
// Remote Query, using protobuf
QueryFactory qf = org.infinispan.client.hotrod.Search.getQueryFactory(remoteCache);
Query<InfinispanObjectEntity> query = qf.create("from nodowntimeupgrade.InfinispanObjectEntity where name = 'searched-name'");
```

# Message definition changes and Ickle queries

Ickle query with a field name that is unknown to the Infinispan server fails.
Therefore, we need to be careful about updates and removals we perform in
message definitions. On top of that, we need to make sure that queries are
constructed with definition changes in mind. For this reason, it may be
necessary to introduce some intermediate storage versions that provide the
migration step on query level. Given the previous example where we replaced
`clientTemplateId` with `clientScopeId`, the migration could be done in the
following steps:

### Query in storage version 1

- Message definition in this version is the original one without `clientScopeId`
  field.
- It is possible to search by `clientTemplateId` without any additional query
  adjustments.

```
FROM nodowntimeupgrade.InfinispanObjectEntity WHERE (clientTemplateId = 'desired-value')
```

### Query in storage version 2

- This is the intermediate version that uses both `clientTemplateId` and
  `clientScopeId`.
- When doing a query it must count on the fact that the storage may contain
  objects with _entity schema version_ `1`, that does not have `clientScopeId`
  defined.
- The Keycloak logical layer searches only by `clientScopeId` attribute.
  Presence of `clientTemplateId` field in the physical layer is hidden from the logical layer.
- Store of version `1` is able to read objects created concurrently by store
  version `2` because these are backward compatible and contain both
  `clientTemplateId` (for version `1`) and `clientScopeId` (for version `2`)
  fields.

For this particular migration path (remember the rule: `clientScopeId =
"template-" + clientTemplateId`), there are two scenarios what the resulting
query looks like based on the searched value.

**Searched value is prefixed with `"template-"`**

In this case, there may be also some object with _entity schema version_ `1`
that fulfills the given criteria; we need to take it into account.

```
FROM nodowntimeupgrade.InfinispanObjectEntity WHERE 
    (entitySchemaVersion >= 2 AND entitySchemaVersion <= 3 AND clientScopeId = 'template-desired-value') 
    OR
    (entitySchemaVersion < 2 AND c.clientTemplateId = 'desired-value')
```

Note the condition `entitySchemaVersion <= 3` comes from the no-downtime
requirement that each version can read all previous versions, but only one
following (`this.version + 1`).


**Searched value is _NOT_ prefixed with `"template-"`**

No adjustment is needed as no object with _entity schema version_ `1` fulfills
this search (`clientScopeId` without `"template-"` prefix is undefined on these
objects).

```
FROM nodowntimeupgrade.InfinispanObjectEntity WHERE (clientScopeId = 'desired-value')
```
### Query in storage version 3

* The message definition must still contain both `clientTemplateId` and
  `clientScopeId` (the old field needs to be present because storage version `2`
  sometimes includes it in Ickle queries).
* There is no migration of Ickle queries anymore, objects with _entity schema
  version_ 1 are not returned from the storage when searching by
  `clientScopeId`.
* If the store detects, on startup, that there are some objects with
  `entitySchemaVersion=1` in the storage, there is a **WARNING** printed that
  warns administrator of possible incomplete search results until they update
  all objects to the _entity schema version_ `>= 2`.

We use only `clientScopeId` in queries.

```
FROM nodowntimeupgrade.InfinispanObjectEntity WHERE (clientScopeId = 'desired-value')
```

### Query in storage in version 4 and beyond

No changes for this particular field in search queries. From the query
perspective, it is possible to remove `clientTemplateId` from the message
definition. Note that this is a breaking change: Retaining presence of this
field is still needed to be able to read older versions of objects and migrating
it to `clientScopeId`. The field definition should be kept as long as there may
be an object of _entity schema version_ lower than `2` stored.


# Speeding up queries with indices

Infinispan uses [`Apache Lucene`](http://lucene.apache.org/) technology to index
values in caches. Entities that contain fields that are indexed are specified in
the cache configuration. Fields that are indexed are then specified in message
definitions. In Java definition, each field that is indexed is annotated with
annotation `@Protodoc`, see the example below.

Cache configuration:
```xml
<indexing>
    <indexed-entities>
        <indexed-entity>nodowntimeupgrade.InfinispanObjectEntity</indexed-entity>
    </indexed-entities>
</indexing>
```

Java message defintion:
```java
@ProtoField(number = 3)
@ProtoDoc("@Field(index = Index.YES, store = Store.YES, analyze = Analyze.YES, analyzer = @Analyzer(definition = \"keyword\"))")
public String name;
```

- **index** - To index or not to index, that is the switch.
- **store** - Index is stored in the host filesystem or the heap. Defaulting to
  the filesystem.
- **analyze** - Configures whether the value is searched as a value or as a
  full-text search. It is also possible to specify an
  [analyzer](https://infinispan.org/docs/stable/titles/developing/developing.html#analysis)
  used. 

# Case-insensitive queries

Often requirement in the Keycloak codebase is to search case-insensitively. For
example, for email addresses we do not want to allow two accounts with emails
`Admin@keycloak.org` and `admin@keycloak.org`. Even though, there is an operator
`LIKE` in Ickle queries, it does not support case-insensitive search. For this
reason, we need to have all fields with this requirement analyzed. 

We need to be also cautious about choosing the correct analyzer. The default
analyzer supports case-insensitive searches, however treats the words in the
text as separate tokens. Therefore, it is not possible to search for strings
with space in it. For example, given a group with name `My group name`, a search
with string `My gr*` would not find any group with default analyzer. It should
be possible to use `keyword` analyzer for searches like this, but it may have
some other disadvantages. This needs to be decided for each index individually.

## Queries with analyzed fields

Ickle queries are
[different](https://infinispan.org/docs/stable/titles/developing/developing.html#using_full_text_search)
for analyzed fields. Operators used for non-analyzed fields do not work for
full-text searches. We need to use `Phrase queries` or `Wildcard queries`. For
example, the following case-insensitive query: `FROM UserEntity ue WHERE
ue.email ILIKE "%@gmail.com"` is replaced with `FROM UserEntity ue WHERE
ue.email : "*@gmail.com"`. For this reason, there are some restrictions on
operator usage for analyzed fields. Supported operators are only ILIKE
(case-insensitive wildcard search), EQ (phrase query without wildcards) and NE
(negated EQ). The set of fields (like e-mail above) that are limited by the
available operators should be clearly documented.


# Transactions

The Keycloak map transaction ensures that:
1. The transaction layer must keep track of all objects returned from the
   storage; all changes done to tracked objects are propagated to the Infinispan
   server in the commit phase
2. To save network utilization, the transaction should be able to decide whether
   there were any changes worth saving performed; some changes may be delayed or
   even irrelevant for the stored state in Infinispan server and should be
   ignored upon commit.
3. The transactional layer participates in the main transaction. It is possible
   to rollback when the Infinispan storing process, or any other participant
   fails.

## Usage of Hot Rod client transaction

The Hot Rod client can participate in [JTA
transactions](https://infinispan.org/docs/stable/titles/hotrod_java/hotrod_java.html#hotrod_transactions).
Unfortunately, the first two items from our requirements are not achievable by
this transaction implementation. Therefore, we need to keep track of changes
done to returned objects manually. The good news is that since `RemoteCache` has
the same interface as `ConcurrentHashMap`, we should be able to use the same
implementation, or maybe just a little bit adjusted implementation of
[`ConcurrentHashMapKeycloakTransaction`](https://github.com/keycloak/keycloak/blob/af3b573d196af882dfb25cdccb98361746e85481/model/map/src/main/java/org/keycloak/models/map/storage/chm/ConcurrentHashMapKeycloakTransaction.java)
for this.

### `TransactionManager` lookup

The Hot Rod client is able to automatically lookup `TransactionManager` using
[`GenericTransactionManagerLookup`](https://docs.jboss.org/infinispan/9.4/apidocs/org/infinispan/transaction/lookup/GenericTransactionManagerLookup.html)
when running in Java EE application server. When there is no
`TransactionManager` running in the container,
[`RemoteTransactionManager`](https://docs.jboss.org/infinispan/10.0/apidocs/org/infinispan/client/hotrod/transaction/manager/RemoteTransactionManager.html)
is returned which can be enlisted to
[KeycloakTransactionManager](https://github.com/keycloak/keycloak/blob/af3b573d196af882dfb25cdccb98361746e85481/server-spi/src/main/java/org/keycloak/models/KeycloakTransactionManager.java).


### Hot Rod client transaction mode

The Hot Rod client transaction can work in the following modes:
- **NONE** - non-transactional - when using this setting, we would not be able
  to achieve point number 3 from our requirements.
- **NON_XA** - in this mode, the `RemoteCache` interacts with the
  `TransactionManager` via
  [`Synchronization`](https://docs.oracle.com/javaee/7/api/javax/transaction/Synchronization.html);
  this means the changes are propagated to the Infinispan storage only when the
  main transaction successfully commits; however, in the case of a failure
  during storing data in the Infinispan server the main transaction is not
  rolled back. 
- **NON_DURABLE_XA** - `RemoteCache` interacts with the `TransactionManager`
  using
  [`XAResource`](https://docs.oracle.com/javaee/7/api/javax/transaction/xa/XAResource.html)
  without possibility of
  [recovery](https://infinispan.org/docs/stable/titles/developing/developing.html#tx_recovery).
- **FULL_XA** - `RemoteCache` interacts with the `TransactionManager` using
  [`XAResource`](https://docs.oracle.com/javaee/7/api/javax/transaction/xa/XAResource.html)
  with the possibility of recovery. Should be avoided due to the [documentation
  note](https://infinispan.org/docs/stable/titles/hotrod_java/hotrod_java.html#hr_transactions_config_server)
  about performance degradation.

Based on the above, **NON_DURABLE_XA** seems to be the best option to use given
the requirements.

# Infinispan Initialization

The Infinispan server will be started externally by an administrator. Based on
the `configureRemoteCaches` option in `HotRodConnectionSpi` configuration,
Keycloak can create remote caches on the Infinispan server using the default
configuration. Some users may want to tweak the default cache configuration
based on their special needs in which case they can create the caches
themselves.

For developers and for testsuite, Keycloak will start an embedded Hot Hod server
and create remote caches.

## Remote caches

Configuration of the remote caches will be stored declaratively in a xml file.
This file will be used either by Keycloak to create remote caches on the
Infinispan server or by an administrator to create them manually.

Example of a remote cache configuration:
```
<infinispan>
   <cache-container>
       ...
       <distributed-cache name="clients" mode="SYNC">
           <encoding media-type="application/x-protostream"/>
       </distributed-cache>
    ...
   </cache-container>
</infinispan>
```

Each map storage type (clients, users, ...) will have a separate remote cache.
The exact configuration of caches to be decided based on the testing. 

## Protobuf schema

It is required to register protobuf schema definition to the Infinispan server
for all entities which will be stored in the caches. We consider two options:
* register Protobuf schemata automatically during the startup by adding them to
  `___protobuf_metadata` cache. We need to make sure that an old schema does not
  override a newer schema if a pod restarts. This is the preferred option.
* `protostream-processor` dependency processes Java annotations in the classes
  at compile time to generate Protobuf schemata. These schemata then can be
  added to the Infinispan server by a user via CLI/REST.

## Changes in cache configuration at runtime

TBD - spin up new Infinispan server with desired cache configuration and migrate
data using the [cache store migrator](https://infinispan.org/docs/dev/titles/upgrading/upgrading.html#offline_data_migration)?


