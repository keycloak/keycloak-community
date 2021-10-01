* **Status**: Draft #1
* **JIRA**: TBD
* **Github Discussion**: [Namespaced Roles](https://github.com/keycloak/keycloak/discussions/8516)

## Motivation

In Keycloak, roles are defined at the Realm and Client levels. These roles can then be assigned directly to a User or to a Group, making every user in that group inherit the role.

This is done through role mapping tables in the database, limiting the way you can assign roles to those two use cases.

We will need more flexibility in the future, for example, to assign a role to a user in the context of a specific group (instead of having everyone in the group inherit a role) or, as the tenancy model gets implemented, a way to both define and assign roles in the context of a tenant.

In order to have this flexibility and still support current use cases, we could create a new kind of role or backport existing roles to this new format: Namespaced roles.

## Specification

A Namespaced Role would follow a certain structure that would unambiguously identify it across the system, following a hierarchical structure.

They would work similarly to ARNs in AWS, where a portion of the format is fixed and always identifiable by position, with the possibility of containing empty fields, containing a final variable part that could be freely named.

An example would be:

`role_v1:/\<realm\>/\<tenant\>/\<client\>/\<group\>/<roleName>`

**Realm**: All resources in Keycloak are namespaced to a realm, so a unique identifier for a role would also contain it.
   - It doesn't support empty values - all roles will be scoped to a realm.  
   - Example: `role_v1:/realm1////roleName`

**Tenant**: There's a proposal to make Keycloak multitenant aware, so even if it's not currently supported, for future compatibility, it should be considered when defining a unique identifier.
   - Supports empty values - A role might be created and assigned at a realm level
   - Example: `role_v1:/realm1/tenantA///roleName`, `role_v1:/realm1///groupA/roleName`

**Client**: Roles can be defined and assigned for a certain client in Keycloak, so a unique identifier has to support this possibility.
   - Supports empty values - A role might be created and assigned without being restricted to a client.
   - Example: `role_v1:/realm1/tenantA/clientId//roleName`, `role_v1:/realm1/tenantA//groupA/roleName`

**Group**: Roles can be assigned to groups or, as intended in this design proposal, assigned to a user in the context of a group.
   - Supports empty values - A role might be created at any level and not necessarily assigned to a group.
   - Example: `role_v1:/realm1/tenantA/clientId/groupB/roleName`, `role_v1:/realm1/tenantA/clientId//roleName`

**RoleName**: Finally, the name of the role that is being defined.
   - Doesn't support empty values - You need to know the name of the role that is being created.
   - Example: `role_v1:/realm1/tenantA/clientId/groupB/roleName`

A possible free-form role identifier could be allowed for resource-server specific roles.

The initial fixed part would be kept in order to identify the role within a realm, tenant, client, group etc, but instead of a fixed name for the role, it would be a reference to other hierarchical role formed with any random structure.

A possible format would be:

- `role_v1:/realm1/tenant1//group1/frole_v1:/mycompany/resources/department-a-roles/developer`

If the role name is another reference to a free form role, starting with `frole_v1:/`, this role will also be represented in a hierarchical way, making it possible for protocol mappers to show them as such in tokens.

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

`role_v1:/realm1/tenant1//iam/manager`
`role_v1:/realm1/tenant1//iam/developer`
`role_v1:/realm1/tenant1//devops/developer`
`role_v1:/realm1/tenant1//devops/devops_role`

Now these roles are assigned directly to `UserA` and `UserB` so that:

- `UserA` has the `role_v1:/realm1/tenant1//iam/manager` and the `role_v1:/realm1/tenant1//devops/developer` roles.
- `UserB` has the `role_v1:/realm1/tenant1//iam/developer` and the `role_v1:/realm1/tenant1//devops/developer` roles.

Also, we should be able to still support giving everyone in a group an inherited role, such as:

- `Group:devops` has the `role_v1:/realm1/tenant1//devops/devops_role` role.

#### Representation in the tokens

So now that we have a namespaced role attached to a user, the representation becomes inherently hierarchical, a **possible representation** in the tokens could look like:

- For UserA

```json
{
    ....
    "roles": {
        "tenants": {
            "tenant1": {
                "groups": {
                    "iam": ["manager"],
                    "devops": ["developer", "devops_role"]
                }
            }
        }
    }
    ....
}

```

- For UserB

```json
{
    ....
    "roles": {
        "tenants": {
            "tenant1": {
                "groups": {
                    "iam": ["developer"],
                    "devops": ["developer", "devops_role"]
                }
            }
        }
    }
    ....
}
```

This also provides a way to isolate certain roles to specific tenants.

- `role_v1:/realm1/tenant1//iam/developer` would only be visible in `tenant1` while
- `role_v1:/realm1/tenant2//iam/somethingelse` would only be visible in `tenant2`.

Or just to specific groups within the realm:

- `role_v1:/realm1///iam/developer` would only be assignable to users in Group `iam`.
- `role_v1:/realm1///devops/devops` would only be assignable to users in Group `devops`.

### Use Case 2 - Custom, resource server specific roles

Another interesting use case is the creation of roles that are not directly restricted to Keycloak's entities such as Realms, Groups, Clients etc.

Right now in Keycloak, clients are effectively a representation of a Resource Server (among other things). You can assign roles to clients and those roles would only be usable on those clients.

As previously mentioned, a role would be namespaced to a certain client such as:

-  `role_v1:/realm1//clientId//roleName`

A different way to be more flexible on giving users the possibility to set their own resource-server specific roles would be to provide a free-form unique role identifier that can be parsed by protocol mappers or just shown in the tokens in a hierarchical way.

For example, a free form role could look like:

- `role_v1:/realm1/tenant1///frole_v1:/mycompany/resources/department-a-roles/developer`

This could map to a token such as:

```json
{
    ....
    "roles": {
        "tenants": {
            "tenant1": {
                "myCompany": {
                   "resources": {
                      "department-a-roles": ["developer"]
                   }
                }
            }
        }
    }
    ....
}
```

#### Advantages of such a structure

Having namespaced claims, following a hierarchical structure, will simplify how we work with roles. And will expand the ways we can assign them to users by giving the possibility to apply roles based on certain contexts.

It would be possible to query a User's roles based on the applied context.

Let's assume that, additionally to previous assigned roles to `UserA`, this user also has the following role in `tenant2`:

- `role_v1:/realm1/tenant2//iam/somethingelse`

Let's say we want to get all the roles for UserA:

- `getRoles("v1", "/")` -> Returns: `role_v1:/realm1/tenant1//iam/manager`, `role_v1:/realm1/tenant1//devops/developer`, `role_v1:/realm1/tenant1//devops/devops_role` and `role_v1:/realm1/tenant2//iam/somethingelse`.

Now, we want to get all the roles that a user has in `tenant1`:

- `getRoles("v1", "/realm1/tenant1")` -> Returns: `role_v1:/realm1/tenant1//iam/manager`, `role_v1:/realm1/tenant1//devops/developer`, `role_v1:/realm1/tenant1//devops/devops_role` 

Or we could go a level further and query by group:

- `getRoles("v1", "/realm1/tenant1//iam")` -> Returns: `role_v1:/realm1/tenant1//iam/manager`.

Or even allow for wildcards and regexps in order to get a certain subset, for example, all roles applied to an `iam` group, regardless of the tenant (Note: this could be useful for the dashboard, but probably restricted when generating tokens)

- `getRoles("v1", "/*/iam")` -> Returns: `role_v1:/realm1/tenant1//iam/manager`, `role_v1:/realm1/tenant2//iam/somethingelse`.

Another advantage is the possibility to show which roles a user has in a specific context in the UI in a tree representation which could be easier to digest.

### UX Improvements when creating and assigning roles

With the hierarchical structure of namespaced roles, the UX to create or assign a role could be improved by showing a tree structure of the current entities in Keycloak.

For example, say you want to create a role `developer` in tenant `TenantB`, group `GroupZ`.

The user could be presented with a screen having a tree structure such as:

```
TenantA
TenantB  
 | 
  -- GroupA
 |
  -- GroupB
 | 
  ...
 |
  -- GroupZ:
     |
      -- RoleA
      -- RoleB
```  
The user could select the GroupZ level and create the role at that level.

The resulting role's unique ID would be:

- `role_v1:/realm1/tenantA//GroupZ/developer`

But the user would not have to get to this level of complexity.

**Note**: Need to find a way to also include clients in this assignment, that is trickier because it doesn't necessarily fit the hierarchical view, so it might need to be a different field that users can choose.

When assigning a role, the same format could be used.

If assigning it to a user, any level could be selected, when assigning it to a group, the tree would be filtered to the level of that group only.


### Integration with Dynamic Scopes / Rich Authorization Requests

One of the main reasons to have this composable way or defining, assigning and referencing roles is to integrate with Dynamic Scopes and RAR in the near future.

For example, when a user requests a dynamic scope `group:iam`, a mapper would be able to get al the roles that a user has in that specific group in a very easy way, by calling querying roles for that specific context:

- `getRoles("v1", "/realm1(current realm)/tenant1(if the group is in a tenant)//iam")` -> Returns: `role_v1:/realm1/tenant1//iam/manager`.

If a user requests a dynamic scope such as: `tenant:tenant1`:

- `getRoles("v1", "/realm1(or current realm)/tenant1")` -> Returns: `role_v1:/realm1/tenant1//iam/manager`, `role_v1:/realm1/tenant1//devops/developer`, `role_v1:/realm1/tenant1//devops/devops_role` 

If the client decided to use free-form roles, such as:

- `role_v1:/realm1/tenant1///frole_v1:/mycompany/resources/department-a-roles/developer`

A possible query for these roles based on dynamic scopes, if the `mycompany/resources` or `mycompany:resources` dynamic scope was provided, the query could look like:

- `getRoles("v1", "/realm1/tenant1///mycompany/resources")` -> Returns: `frole_v1:/mycompany/resources/department-a-roles/developer` 


### Implementation details

Roles are a core entity within Keycloak and making changes to them have the potential to break significant parts of the code and functionality.

An incremental way to implement namespaced roles would be to add a new field `namespace` in the `keycloak_role` database table, containig the whole id string of the role.

Checks would need to be made to make sure that:

- The realm in the namespace matches the role's realm database field.
- On role mappings, if the namespace field is present, the following checks could be made:

  - On user-role mapping: Make sure the user belongs to the group variable in the namespace, if present, same thing with tenants whenever we have them etc.

  - On group-role mapping: Make sure the group in the namespace is the same as the group being mapped to the role.

  - On client-role mapping: Make sure the client variable in the namespace matches the client that the role is being mapped to.
- On resource deletion: The system should query all roles with the resource in its namespace and delete them.
  - Example: Role `role_v1:/realm1///groupA/developer` exists. User deletes `groupA`. The system will query `getRoles("v1", "/realm1///groupA")` and delete any returned entries. 

### Extensibility

Protocol Mappers would be able to work with a predefined role id format to create any necessary representation in tokens.

Other mappers could be defined to work with the free form roles formats by receiving configuration on how to access and map them to specific fields in the tokens.

In case a new base entity needs to be added to the fixed part of the role identifier, a new namespace version could be defined such as:

- `role_v2:/\<realm\>/\<tenant\>/\<newField\>/\<client\>/\<group\>/\<role\>`

And the hypothetical roles query would receive the new version as a parameter:

`getRoles("v2", "/realm1//newField")`

When creating a role, users could choose which version to use and configure new mappers to either or all versions depending on the query they run.