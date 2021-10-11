* **Status**: Draft #3
* **Github Discussion**: [Namespaced Roles](https://github.com/keycloak/keycloak/discussions/8516)

**Note**: Still trying to find a good fragment name for User Managed roles (`/ud/`).

## Motivation

In an attempt to improve Keycloak's authorization capabilities, both for Keycloak administration and for user defined controls, we need to improve how Roles are defined, simplifying some use cases to get rid of unnecessary structures and to flexibilize how users can work with roles in the system.

Some of the proposed changes are:

- Remove clients that group realm specific roles.

- Allow users to create a set of roles that can be shared among cients and groups.

- Allow users to define their own role namespaces to match their organizational structure and business needs.

In order to have this flexibility and still support current use cases, we could create a new kind of role or backport existing roles to this new format: Namespaced roles.

## Specification

A Namespaced Role would be defined as a [Unified Resource Identifier (URI)](https://datatracker.ietf.org/doc/html/rfc3986) whose segments would identify the kinds and values of the entities that the roles are scoped to.

The formal syntax would look like:

- `/(kc|ud|"")/({entity-type}/{entity-name})*/{role-name}`

The first segment would let the system identify the kind of role. The `/kc/` fragment would be restricted to Keycloak administrative roles, `/ud/` would identify roles managed by users.

If the namespace doesn't contain any of these segments, the system would interpret this role as a custom, user defined namespace.

A few examples would be:

`/kc/realms/realm-a/view-clients`

`/ud/groups/group-a/admin`

`/my-company/my-business-unit/my-cost-center/my-role`

Keycloak would parse these roles by looking at the entity types before the variables in order to query and assign them.

## Keycloak Admin Roles

From Keycloak documentation

```
Admin users within the master realm can be granted management privileges to one or more other realms in the system. Each realm in {project_name} is represented by a client in the master realm. The name of the client is <realm name>-realm. These clients each have client-level roles defined which define varying level of access to manage an individual realm.
```
Namespaced Roles would make it possible to deprecate these clients whose only purpose is to group management roles together.

For example, when a new realm `Realm-A` is created, a client called `realm-a-realm` is created in the `Master` realm with these management roles.

Additionally, a client called `realm-management` is created within the new realm with the same roles.

These clients serve no purpose other than grouping these roles together, so they could be replaced by creating a specific namespace.

The `realm-a-realm` and the Realm-A's `realm-management` clients could be replaced by defining the following namespace:

- `/kc/realms/realm-a/{role-name}`

In order to give `UserA` the `view-clients` role in `Realm-A`, the following namespaced role could be defined and assigned:

- `/kc/realms/realm-a/view-clients`

Global admin role(s) could be defined with the following format:

- `/kc/admin/{role-name}`

### Fine-grained management permissions

Rigt now fine-grained management permissions are defined in Keycloak making use of Authorization Services and Policies.

A possible alternative would be defining Keycloak admin namespaces with the entity types and names that want to be managed.

For example, to manage a specific client:

- `/kc/clients/{client-id}/manage`

Restrict User Role Mappings:

- `/kc/clients/{client-id}/map-roles`

Managing the membership of a group:

- `/kc/groups/{group-name}/manage-membership`

And a large number of use cases that are contemplated in https://www.keycloak.org/docs/latest/server_admin/index.html#_fine_grain_permissions.

## User Managed Roles

These roles would have a meaning for Users but would still be understood by Keycloak.

These roles would allow users to define roles in the context of specific entities, but unlike the Keycloak restricted management roles (`/kc/`), these could eventually end up in tokens through Protocol Mappers or used for policies in Authorization Services.

These roles would start with the `/ud/` segment. For example:

- `/ud/groups/{group-name}/{role-name}` -> Roles that can only be assigned in the context of a specific group.

- `/ud/clients/{client-id}/{role-name}` -> Roles that can only be assigned in the context of a spefici client.

- `/ud/{role-name}` -> Roles without any scope other than the `/ud/` namespace. These would replace the current realm roles.

The current use cases would still be supported, such as defining a simple role that can be mapped to a User or a Group, but additional scoping could be provided by adding certain segments to the namespaces.

### Use Case - Assign a role in the context of a group

Say we want to have the following group composition in our realm, most likely imported from a federated LDAP:

realm/realm1
 - group:iam
 - group:devops
 - group:leadership

Following the current implementation, the users would need to define roles at the realm level and then assign them to these groups. Everyone in those groups would inherit the roles.

But for a user, these groups may have business meaning or reflect organizational structure within their company, they aren't just a mean to group users together with the same roles, but rather fully functional teams that users can be part of with different roles.

So UserA would be a `manager` in `group:iam`, but UserB could be a `developer`in `group:iam`.

Namespaced roles would be created and loosely assigned based on the group, so let's say we have the following roles:

`/ud/groups/iam/manager`
`/ud/groups/iam/developer`
`/ud/groups/devops/developer`
`/ud/groups/devops/devops_role`

Now these roles are assigned directly to `UserA` and `UserB` so that:

- `UserA` has the `/ud/groups/iam/manager` and the `/ud/groups/devops/developer` roles.
- `UserB` has the `/ud/groups/iam/developer` and the `/ud/groups/devops/developer` roles.

Also, we should be able to still support giving everyone in a group an inherited role, such as:

- Giving `Group:devops` the `/ud/groups/devops/devops_role` role or without the groups namespace fragment `/ud/devops_role`.

#### Representation in the tokens

So now that we have a namespaced role attached to a user, a **possible representation** in the tokens, defined by a token mappers could look like:

- For UserA

```json
{
    ....
    "roles": {
        "groups": {
            "iam": ["manager"],
            "devops": ["developer", "devops_role"]
        }
    }
    ....
}

```

As mappers can be custom implementations, users would be able to define other representations, or Keycloak could provide more built-in mappers to do so. 

An alternative representation could look like:

```json
{
    ....
    "roles": [
        "groups/iam/manager",
        "groups/devops/developer",
        "groups/devops/devops_role"
    ]
    ....
}

```

- For UserB

```json
{
    ....
    "roles": {
        "groups": {
            "iam": ["developer"],
            "devops": ["developer", "devops_role"]
        }
    }
    ....
}
```

The whole idea is that by leveraging namespaces, users can define the structure that they need for their business needs.

### Use Case 2 - Isolate roles to specific entities

It will also be possible to isole certain roles to specific entities such as groups or clients.

- `/ud/groups/groupA/developer` would only be assignable in `group:GroupA` while
- `/ud/groups/groupB/somethingelse` would only be assignable in `group:GroupB`.


## User Defined Roles

A free-form role identifier would be allowed for resource-server specific roles thay may hold business meaning for users. It would lack the fixed `/kc/` or `/ud/` segments and allow any user defined path. For example:

- `/business-units/myBusinessUnit/const-centers/myCostCenter/roleName`

**Question**: In the initial proposal, these roles would be preceded by a fixed part that would identify the context where this role would be applied. We could do somethng similar here such as:

- `/groups/{group-name}/role-path/business-units/myBusinessUnit/const-centers/myCostCenter/roleName`

But not convinced on the way to let Keycloak know that that's not a role name but rather a new namespace.

### Representation in tokens

Let's say a user defines the following namespaced role

- `/mycompany/resources/department-a-roles/developer`

This could map to a token such as:

```json
{
    ....
    "roles": {
        "myCompany": {
            "resources": {
                "department-a-roles": ["developer"]
            }
        }
    }
    ....
}
```

### Using Namespace references instead of specific roles

Having to assign a lot of roles in a specific namespace one by one would be a big user experience nightmare, so we'll need a way to assign a role namespace itself instead of individual roles within it.

Let's say we defined a namespace such as `/custom-roles-namespace/` with a few roles such as `developer`, `admin` and `devops`.

Following the specification, these roles would be defined such as:

- `/custom-roles-namespace/developer`
- `/custom-roles-namespace/admin`
- `/custom-roles-namespace/devops`

Maybe we want to assign all these roles to a User or a Group. The UI can allow users to assign full namespaces.

- `mapRole("/custom-roles-namespace/#", "subject-id")` -> Keycloak would query all the roles contained in that namespace and individually assign all these roles to the subjects.

#### Advantages of such a structure

Having namespaced claims, will simplify how we work with roles. And will expand the ways we can assign them to users by giving the possibility to apply roles based on certain contexts.

It would be possible to query a User's roles based on the applied context.

We can define the following methods:

- `getKeycloakManagementRoles(String roleURI)` -> Queries the database for roles within the Keycloak managed namespace (`/kc/`).

- `getUserManagedRoles(String roleURI)` -> Queries the database for roles within the user managed namespace (`/ud/`).

- `getCustomRoles(String roleURI)` -> Queries the database for roles outside of the Keycloak or User managed namespaces (`/kc/` and `/ud/`).

Let's say we want to get all the roles for UserA:

- `getUserManagedRoles("/")` -> Returns: `/ud/groups/iam/manager`, `/ud/groups/devops/developer` and `/ud/groups/devops/devops_role`.

Now, we want to get all the roles that a user has in `/groups/devops`:

- `getUserManagedRoles("/groups/devops")` -> Returns: `/ud/groups/iam/manager`, `/ud/groups/devops/developer`, `/ud/groups/devops/devops_role` 

We may also need to know whether a user can manage a realm:

- `getKeycloakManagementRoles("/realms/realm-a/manage-realm")`.

### UX Improvements when creating and assigning roles

When creating namespaced roles, the UI could help users place a group in a specific context that would build a role namespace in a visual way.

A User wants to create the role `developer` in group `GroupZ`.

The UI could ask the user to add layers to the selection by showing some predefined structure. Asking the user repeatedly to add more context to the namespace, if possible.

For example, it would make sense to ask for extra context if the first one was at a Realm level, but it wouldn't probably ask for extra context if the previous selection was a group.

During this time, the role namespace would build automatically and show the user the final role namespace.

The resulting role namespace for this use case would be:

- `/ud/groups/GroupZ/developer`

But the user would not have to get to this level of complexity.

### Integration with Dynamic Scopes / Rich Authorization Requests

One of the main reasons to have this composable way or defining, assigning and referencing roles is to integrate with Dynamic Scopes and RAR in the near future.

For example, when a user requests a dynamic scope `group:iam`, a mapper would be able to get al the roles that a user has in that specific group in a very easy way, by calling querying roles for that specific context:

- `getUserManagedRoles("/groups/iam")` -> Returns: `/ud/groups/iam/manager`.

If the client decided to use free-form roles, such as:

- `/mycompany/resources/department-a-roles/developer`

A possible query for these roles based on dynamic scopes, if the `mycompany/resources` or `mycompany:resources` dynamic scope was provided, the query could look like:

- `getCustomRoles("/mycompany/resources")` -> Returns: `/mycompany/resources/department-a-roles/developer` 

### Implementation details

Roles are a core entity within Keycloak and making changes to them have the potential to break significant parts of the code and functionality.

An incremental way to implement namespaced roles would be to add a new field `namespace` in the `keycloak_role` database table, containig the whole namespace URI string of the role.

Checks would need to be made to make sure that:

- On role mappings, if the namespace field is present, the following checks could be made:

  - On user-role mapping: Make sure the user belongs to the group variable in the namespace, if present.

  - On group-role mapping: Make sure the group in the namespace is the same as the group being mapped to the role.

  - On client-role mapping: Make sure the client variable in the namespace matches the client that the role is being mapped to.
  
- On resource deletion: The system should query all roles with the resource in its namespace and delete them.
  - Example: Role `/ud/groups/groupA/developer` exists. User deletes `groupA`. The system will query `getUserManagedRoles("/groups/groupA")` and delete any returned entries.

#### Migration

Current roles could potentially be migrated to a structure that resembles a unscoped User Managed Roles.

For example, a Realm level role such as `developer` could be migrated to `/ud/developer` by adding the `/ud/` fragment the the `namespace` database field for the migrated role.

A Client level role such as `uma_protection` in client with ID 123-123-123-123 could be migrated to `/ud/clients/123-123-123-123/uma_protection` by adding the `/ud/clients/123-123-123-123/` URI to the `namespace` database field for the migrated role.

Additionally, Keycloak management roles could follow the same pattern.

The role `view-users` created in the `realm-management` client for realm `RealmB` can be migrated to `/kc/realms/RealmB/view-users`.

### Possible future additions

#### Tenants

We are working on a proposal that will enable multi-tenancy for Keycloak. 

This specification considers that there may be new entities being added to Keycloak that would need to be added to role namespaces, for direct or hierarchical role scoping.

These are some use cases where tenants could be considered:

- Create a role to manage a specific tenant:
   - `/kc/tenants/tenant-a/manage-tenant`

- Create a role scoped to a group, but within a specific tenant:
  - `/ud/tenants/tenant-a/groups/groupA/groupScopedRole`

When we have a hierarchical structure such as the previous one, we could also allow for wildcard queries:

- `getUserManagedRoles("*/groups/iam")` -> Returns all roles for any group called IAM in any tenant.

#### Namespaced Custom Roles

The initial implementation of Custom Roles wouldn't contain a context or scope within their URI.

A Role would be defined such as:

- `/some/free-form/namespace/uri/role`

With this format, it's impossible to restrict these roles or to add them to individual users within a group's context, so in the future, we may support adding an additional namespace URI such as:

- `/ud/groups/groupB/cRole:/some/free-form/namespace/uri/role`

This way we could scope these custom roles to specific contexts just like we'd do with User Manged and Keycloak Managed roles.

#### Role Versions

In the initial iteration, roles namespaces wouldn't contain any version, but it would be benefitial to add them in case we need to change the structure or behaviour in the future.

We could treat existing roles as v1 and add a new URI fragment to define a version:

- `role_v2:/(kc|ud|"")/({entity-type}/{entity-name})*/{role-name}` or `/version/v2/(kc|ud|"")/({entity-type}/{entity-name})*/{role-name}`