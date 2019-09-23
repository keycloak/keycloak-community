# User Profile

* **Status**: Notes
* **JIRA**: [KEYCLOAK-2966](https://issues.jboss.org/browse/KEYCLOAK-2966)


## Motivation

Keycloak does not currently have a way to define a user profile directly. There is also no single place to define 
validation of a user when it is created or updated.

Having a single place where a user profile is defined for a realm would enable:

* Dynamic registration form
* Dynamic user profile form in account and admin consoles
* Proper validation of user profiles
* Ability to select user attributes in mappers
* Requesting additional information from users as needed


## User Profile SPI

User Profile for a realm should be handled by a User Profile provider, making it possible to customize how user profiles
are handled by the system.

A User Profile provider should be able to list the attributes for the user profile as well as validate creation
and updates to a user.

UserProfileProvider:

* UserProfile getUserProfile(RealmModel realm)
* boolean validate(UserProfileEvent event)


## Default User Profile provider

New realms will be configured to use the default user profile provider, which will have a basic initial configuration.

The built-in user profile provider should support defining the user profile in JSON. An example would be:

    {
        "attributes": [{
            "key": "firstname",
            "view": ["admin", "user"],
            "edit": ["admin", "user"],
            "required":"never|admin|user|always|scopes:["scope"]"
            "validation": {
                "regex":"[A-Za-z]{1,100}"
            }
        }
    }
    

### UI

The admin console will initially have a textarea field to edit the user profile, but will eventually have a UI to
manage the profile.


## Legacy User Profile provider

A provider that emulates the previous behavior for Keycloak. During upgrades existing realms will be configured to use 
the legacy user profile providers.

## Wiring

### Registration
### Account console
### Admin console
### Actions
### User federation
### Identity brokering
### Mappers