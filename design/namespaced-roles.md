* **Status**: Draft #1
* **JIRA**: TBD
* **Github Discussion**: [Namespaced Roles](https://github.com/keycloak/keycloak/discussions/8516)

## Motivation

In Keycloak, roles are defined at the Realm and Client levels. These roles can then be assigned directly to a User or to a Group, making every user in that group inherit the role.

This is done through role mapping tables in the database, limiting the way you can assign roles to those two use cases.

We will need more flexibility in the future, for example, to assign a role to a user in the context of a specific group (instead of having everyone in the group inherit a role) or, as the tenancy model gets implemented, a way to both define and assign roles in the context of a tenant.

A few things that we'd also like to change in Keycloak with regard to roles are:

- Avoid the need to create clients to "namespace" roles for specific cases. Right now when creating a new Realm, Keycloak creates some default clients such as the `Realm Management` one that holds Keycloak specific roles to work with the dashboard.

- The ability to share a group of roles among clients instead of having to define them individually per client.

- Improve the experience of managing roles. By namespacing them, it would be easier to add custom structures that have business meaning for users.

In order to have this flexibility and still support current use cases, we could create a new kind of role or backport existing roles to this new format: Namespaced roles.

## Specification

A Namespaced Role would contain a version, a path that start with a known particle to identify the nature of the role, and finally the role name.

For example, Keycloak management specific roles would start with `/kc`.

User defined roles, but still used to interact with Keycloak entities would start with `/ud`.

Totally custom namespaces created with a specific business meaning for users wouldn't contain any of the previous particles.

Namespaced Roles would always follow a structure that contains a fixed part identifying the entity, followed by a variable part containing its identifier. This pattern could be repeated in a hierarchical way.

A few examples would be:

`role_v1:/kc/groups/{group-name}/roleName`

`role_v1:/ud/tenants/{tenant-name}/roleName`

`role_v1:/ud/tenants/{tenant-name}/clients/{client-name}/roleName`

Some of the Keycloak supported entities would be:

**Tenant**: There's a proposal to make Keycloak multitenant aware, so even if it's not currently supported, for future compatibility, it should be considered when defining a unique identifier.
   - Example: `role_v1:/kc/tenants/tenantA/roleName`.

**Client**: Roles can be defined and assigned for a certain client in Keycloak, so a unique identifier has to support this possibility.
   - Example: `role_v1:/kc/clients/clientId/roleName`.

Alternatively, if a namespace wants to be created for roles that can be shared among clients, it could be defined such as:  `role_v1:/clients/*/roleName`.

**Group**: Roles can be assigned to groups or, as intended in this design proposal, assigned to a user in the context of a group.
   - Example: `role_v1:/groups/groupA/roleName`.

Alternatively, if a namespace wants to be created for roles that can be shared among groups, it could be defined such as:  `role_v1:/groups/*/roleName`

A free-form role identifier would be allowed for resource-server specific roles thay may hold business meaning for users. It would lack the fixed `kc` or `ud` particles and allow any user defined path. For example:

- `role_v1:/business-units/myBusinessUnit/const-centers/myCostCenter/roleName`

**Question**: In the initial proposal, these roles would be preceded by a fixed part that would identify the context where this role would be applied. We could do somethng similar here such as:

- `role_v1:/groups/{group-name}/role-path/business-units/myBusinessUnit/const-centers/myCostCenter/roleName`

But not convinced on the way to let Keycloak know that that's not a role name but rather a new namespace.

## Management Roles

These roles would only exist in the Master realm and would have a meaning for Keycloak, hence the `kc` fragment.

- `role_v1:/kc/admin/{role-name}` -> Roles to manage the Keycloak instance.

When a new Realm is created in Keycloak, a new client is created in the Master realm to represent a namespace for this new Realm's admin roles. With the exception of the previous one, they will be used to deprecate the Clients that Keycloak creates to group certain management roles together.

The rest of these roles would always be scoped to a specific realm that would replace the current client-per-realm.

- `role_v1:/kc/realms/{realm-name}/clients/{client-name}/{role-name}` -> Roles to manage a certain client.
- `role_v1:/kc/realms/{realm-name}/groups/{group-name}/{role-name}` -> Roles to manage a certain group.
- `role_v1:/kc/realms/{realm-name}/tenants/{tenant-name}/{role-name}` -> Roles to manage a certain tenant.
- `role_v1:/kc/realms/{realm-name}/tenants/{tenant-name}/groups/{group-name}/{role-name}` -> Roles to manage a certain group in a certain tenant.

These realm-clients that are created in the Master realm include roles such as:

- manage-realm, manage-users, manage-authorization, etc.

We would replace these clients by creating the following namespace:

- `role_v1:/kc/realms/{new-realm-name}/{role-name}`

## User Defined Roles

These roles would have a meaning for Users, hence the `ud (user defined)` fragment.

The same format, but defined by users to map roles to these entities would look like:

- `role_v1:/ud/groups/{group-name}/{role-name}` -> Roles that can only be assigned in the context of a specific group.
- `role_v1:/ud/tenants/{tenant-name}/{role-name}` -> Roles that can only be assigned in the context of a specific tenant.
- `role_v1:/ud/tenants/{tenant-name}/groups/{group-name}/{role-name}` -> Roles that can only be assigned in the context of a specific tenant and group.

Keycloak would parse these roles by looking at the entity types before the variables in order to query and assign them.

## Use Cases

As previously mentioned, there are a few use cases that we'll want to support and are currently not possible in Keycloak.

### Use Case 1 - Assign a role in the context of a group

Say we want to have the following group composition in our realm (or our tenant, in the future), most likely imported from a federated LDAP:

realm/tenant:tenant1
 - group:iam
 - group:devops
 - group:leadership

Following the current implementation, the users would need to define roles at the realm level and then assign them to these groups. Everyone in those groups would inherit the roles.

But for a client, these groups may have business meaning or reflect organizational structure within their company, they aren't just a mean to group users together with the same roles, but rather fully functional teams that users can be part of with different roles.

So UserA would be a `manager` in `group:iam`, but UserB could be a `developer`in `group:iam`.

Namespaced roles would be specified when created, and loosely assigned based on the namespace, so let's say we have the following roles:

`role_v1:/ud/groups/iam/manager`
`role_v1:/ud/groups/iam/developer`
`role_v1:/ud/groups/devops/developer`
`role_v1:/ud/groups/devops/devops_role`

Now these roles are assigned directly to `UserA` and `UserB` so that:

- `UserA` has the `role_v1:/ud/groups/iam/manager` and the `role_v1:/ud/groups/devops/developer` roles.
- `UserB` has the `role_v1:/ud/groups/iam/developer` and the `role_v1:/ud/groups/devops/developer` roles.

Also, we should be able to still support giving everyone in a group an inherited role, such as:

- `Group:devops` has the `role_v1:/ud/groups/devops/devops_role` role or `role_v1:/ud/devops_role`.

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

It will also be possible to isole certain roles to specific entities such as tenants.

- `role_v1:/ud/tenants/tenant1/developer` would only be assignable in `tenant1` while
- `role_v1:/ud/tenants/tenant2/somethingelse` would only be assignable in `tenant2`.

Or just to specific clients within the realm:

- `role_v1:/kc/clients/client-a/client-admin` would only give users the `client-admin` role when interacting with the `client-a` client.
- `role_v1:/kc/groups/devops/group-admin` would only give users the `group-admin` role when interacting with with the `devops` groups.

### Use Case 3 - Custom, resource server specific roles

Another interesting use case is the creation of roles that are not directly restricted to Keycloak's entities such as Realms, Groups, Clients etc.

Right now in Keycloak, clients are effectively a representation of a Resource Server (among other things). You can assign roles to clients and those roles would only be usable on those clients.

As previously mentioned, a role would be namespaced to a certain client such as:

-  `role_v1:/ud/clients/{client-name}/{role-name}`

or with a regexp, they could be namespaced to a specific subset of clients:

- `role_v1:/ud/clients/{regexp-containing-wildcard}/{role-name}`

A different way to be even more flexible by giving users the possibility to set their own resource-server specific would be using the custom namespaced roles.

For example, a free form role could look like:

- `role_v1:/mycompany/resources/department-a-roles/developer`

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
Or even scope this custom namespace to a specific client or client sets:

- `role_v1:/ud/clients/{client-name-or-regexp}/frole:/mycompany/resources/department-a-roles/developer`

So this role can only be mapped to a token issued by those specific clients.

#### Advantages of such a structure

Having namespaced claims, will simplify how we work with roles. And will expand the ways we can assign them to users by giving the possibility to apply roles based on certain contexts.

It would be possible to query a User's roles based on the applied context.

Let's assume that, additionally to previous assigned roles to `UserA`, this user also has the following role in `tenant2`:

- `role_v1:/ud/tenants/tenant2/groups/iam/somethingelse`

Let's say we want to get all the roles for UserA:

- `getRoles("v1", "ud","/")` -> Returns: `role_v1:/ud/groups/iam/manager`, `role_v1:/ud/groups/devops/developer`, `role_v1:/ud/groups/devops/devops_role` and `role_v1:/ud/tenants/tenant2/groups/iam/somethingelse`.

Now, we want to get all the roles that a user has in `/groups/devops`:

- `getRoles("v1", "ud", "/groups/devops")` -> Returns: `role_v1:/ud/groups/iam/manager`, `role_v1:/ud/groups/devops/developer`, `role_v1:/ud/groups/devops/devops_role` 

Or we could filter by tenant:

- `getRoles("v1", "ud", "/tenants/tenant2")` -> Returns: `role_v1:/ud/tenants/tenant2/groups/iam/somethingelse`.

Or even allow for wildcards and regexps in order to get a certain subset, for example, all roles applied to an `iam` group, regardless of the tenant (Note: this could be useful for the dashboard, but probably restricted when generating tokens)

- `getRoles("v1", "ud", "*/groups/iam")` -> Returns: `role_v1:/ud/groups/iam/manager`, `role_v1:/ud/tenants/tenant2/groups/iam/somethingelse`.

### UX Improvements when creating and assigning roles

When creating namespaced roles, the UI could help users place a group in a specific context that would build a role namespace in a visual way.

A User wants to create the role `developer` in tenant `TenantB`, group `GroupZ`.

The UI could ask the user to add layers to the selection by showing some predefined structure:

```
What would be the root of the role namespace?
Choose between: Realm, Tenant, Client or Group.

::: User Selects TenantB

Want to add more contexts to the role?
As Tenant was previously selected, the UI would show: Client or Group.
```

During this time, the role namespace would build automatically and show the user the final role namespace.

The resulting role namespace for this use case would be:

- `role_v1:/ud/tenants/tenantB/groups/GroupZ/developer`

But the user would not have to get to this level of complexity.

### Integration with Dynamic Scopes / Rich Authorization Requests

One of the main reasons to have this composable way or defining, assigning and referencing roles is to integrate with Dynamic Scopes and RAR in the near future.

For example, when a user requests a dynamic scope `group:iam`, a mapper would be able to get al the roles that a user has in that specific group in a very easy way, by calling querying roles for that specific context:

- `getRoles("v1", "ud", "/groups/iam")` -> Returns: `role_v1:/ud/groups/iam/manager`.

If a user requests a dynamic scope such as: `tenant:tenant1`:

- `getRoles("v1", "ud", "/tenants/tenant1")` -> Returns: `role_v1:/ud/tenants/tenant1/groups/iam/manager`, `role_v1:/ud/tenants/tenant1/groups/devops/developer`, `role_v1:/ud/tenants/tenant1/groups/devops/devops_role` 

If the client decided to use free-form roles, such as:

- `role_v1:/mycompany/resources/department-a-roles/developer`

A possible query for these roles based on dynamic scopes, if the `mycompany/resources` or `mycompany:resources` dynamic scope was provided, the query could look like:

- `getRoles("v1", "", "/mycompany/resources")` -> Returns: `role_v1:/mycompany/resources/department-a-roles/developer` 


### Implementation details

Roles are a core entity within Keycloak and making changes to them have the potential to break significant parts of the code and functionality.

An incremental way to implement namespaced roles would be to add a new field `namespace` in the `keycloak_role` database table, containig the whole id string of the role.

Checks would need to be made to make sure that:

- On role mappings, if the namespace field is present, the following checks could be made:

  - On user-role mapping: Make sure the user belongs to the group variable in the namespace, if present, same thing with tenants whenever we have them etc.

  - On group-role mapping: Make sure the group in the namespace is the same as the group being mapped to the role.

  - On client-role mapping: Make sure the client variable in the namespace matches the client that the role is being mapped to.
  
- On resource deletion: The system should query all roles with the resource in its namespace and delete them.
  - Example: Role `role_v1:/ud/groups/groupA/developer` exists. User deletes `groupA`. The system will query `getRoles("v1", "ud, "/groups/groupA")` and delete any returned entries. 

### Extensibility

Protocol Mappers would be able to work with preconfigured role namespace formats to create any necessary representation in tokens.

In case we need to evolve these roles in a non backwards compatibility way, we can start creating roles with the breaking change with a different role version preffix:

- `role_v2:{any-possible-change}`

And the hypothetical roles query would receive the new version as a parameter:

`getRoles("v2", "ud", {new-query})`

When creating a role, users could choose which version to use and configure new mappers to either or all versions depending on the query they run.