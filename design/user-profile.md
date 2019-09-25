# User Profile

* **Status**: Notes
* **JIRA**: [KEYCLOAK-2966](https://issues.jboss.org/browse/KEYCLOAK-2966)


## Motivation

Keycloak does not currently have a way to define a user profile directly. There is also no single place to define validation of a user when the user is created or updated.

With a defined user profile it would be possible to:

* Define validation in a single place and properly validate all updates to user profiles
* Show and hide attributes on the registration form
* Select user attributes in mappers instead of supplying the key
* Request missing information from users on demand


## User Profile SPI

To allow user profiles to be customized an SPI will be introduced. 

The User Profile provider will be responsible for providing metadata around user profiles as well as validation of user profiles. The API will not be covered in this design, but rather only the requirements for the API.

The User Profile provider needs to be supplied with the following context when validating:

* Is it a new user or update to existing user?
* What is the source of the information (user, admin, user federation, identity brokering)
* What scopes if any are being requested?

If validation is not passed the following information needs to be available from the provider:

* List of attributes that are not valid, with an error message per attribute
* List of missing attributes
* List of unsupported attributes

It should also be possible to only validate a set of attributes. An example use-case for this is an action that is asking for a single attribute, but is not able to request additional attributes if missing.

The user profile metadata provided by the provider should be able to answer the following questions within the context of the request:

* Is a specific attribute required?
* Is a specific attribute supported?
* List of supported attributes


## Legacy Provider

For backwards compatibility a legacy provider will be implemented that hardcodes the current behavior of Keycloak. Existing realms will be configured to use the legacy provider.

The legacy provider will:

* Support firstname, lastname and email - validation only for email
* Ignore unknown attributes - current behavior is that additional attributes needs to be validated elsewhere (i.e. through custom actions)


## Default Provider

The default provider will support defining a user attribute as a JSON document. Initially the admin console will simply allow editing the JSON directly, but the plan is to introduce an editor.

For new realms the this will be used as the default provider. There will be a hardcoded user profile (see separate section on user profile attributes). A realm will use the hardcoded user profile unless one is configured specifically for the realm.

The user profile will be saved as JSON in a realm attribute. When editing the user profile for a realm the default hardcoded user profile will be shown, but it will only be saved in the realm attribute if the profile is indeed modified and not equal to the built-in profile. 

The JSON document will consist of a list of user attributes, where each attribute will contain fields for:

* Key - the attribute key
* Validation - a list of validators, including config
* Required - ability to specify when the attribute is required, more below
* Can the user view the attribute? Can the admin view the attribute?
* Can the user edit the attribute? Can the admin edit the attribute?
* Label to use when displayed in forms (to be discussed, see Dynamic Forms section)
* UI order (to be discussed, see Dynamic Forms section)
* Input type (to be discussed, see Dynamic Forms section)

Ability to specify when an attribute is required should be quite flexible to cover use-cases like:

* Don't ask user for D.O.B. until a client requests the scope "dob"
* Allow an admin to create an initial user with only specifying email, then requiring user to fill in first and last names on first login
* Ask user for missing information after login with a social provider

A very rough idea on how the user profile JSON could look like:

    {
        "attributes": [{
            "key": "department",
            "view": ["admin", "user"],
            "edit": ["admin"],
            "requirement":"always",
            "validation": {
                "regex":"[A-Za-z]{1,100}"
            }
        }
    }

Requirement should support following values:

* `optional` (default if no value given)
* `always`
* `user`
* `{ "scope" : "scope-1" }`
* `{ "scope" : [ "scope-1", "scope-2" ] }`

The `user` requirement allows an admin to create a user without specifying the value, while the user will then be required to enter the value on first login.

The `scope` requirement allows an attribute to only be required when a specific client scope is being requested.

Validation should support the following values:

* `validator-id`
* `{ "validator-id" : "<config>" }`
* `{ "validator-id" : { ... } }`

This will allow using built-in validators as well as adding custom validators. A validator can take an option config either as a single field or a JSON object.

Built-in validators should cover:

* Name
* Email
* Pattern
* Number, with min and max
* Length, with min and max


## Built-in Attributes

Keycloak should add more attributes out of the box. This means defining the standard key for the attribute as well as optionally rendering in forms.

The lists should be based on standard [claims from OpenID Connect Core](https://openid.net/specs/openid-connect-core-1_0.html#StandardClaims), but not covering all attributes at least not initially.


## Dynamic Forms

With regards to forms there is two options we can choose from:

* Crafted forms - craft forms such as registration form, where fields are shown or hidden based on user profile. To add custom attributes the form would be extended.
* Dynamic forms - add additional metadata to user profile attributes so forms can be dynamically generated.


## Wiring

This section will discuss how the user profile will be used within Keycloak. 

### Registration

The default registration form will display fields as defined in the user profile.

It will also validate the profile and display error messages on each attribute.

### Actions

The update profile action will be extended to support asking for only attributes that are missing or invalid.

### Account Console

The new Account Console will display fields as defined the user profile, as well as use the profile to validate and display individual messages.

The old Account Console will require custom forms to add additional fields, it will use the profile to validate, but will not display individual messages.

### Admin Console

The user profile will display fields as defined in the user profile. Adding additional attributes will only allow selecting attributes from the profile and not add arbritary attributes. 

### User Federation

TBD

### Identity Brokering

TBD

### Mappers

All mappers that use user attributes should provide a drop-down with list of attributes and no longer have a input field to specify the attribute.


## Milestones

### M1 Validation

* Define SPI
* Add legacy provider
* Validate self-registration form
* Validate updates to profile through old Account Console
* Validate updates to profile through Account REST and new Account Console

### M2 Default Provider

* Add default provider
* Profile defined through text-area in Admin Console
* Ability to set User Profile provider for realm

### M3 Dynamic forms

* Extend registration form, with fields conditionally displayed based on user profile
* Extend profile form in new Account Console, with fields conditionally displayed based on user profile

### M4 Admin Console

* Validate user creation in admin console
* Validate user updates in admin console
* Extend user form in admin console, with fields conditionally displayed based on user profile
* Merge user profile and attributes forms in order to create user to pass validation

### M5 User Profile editor

* Editor in admin console for defining user profile

### M6 Mappers

* Select user attributes in mappers

### M7 User Federation & Identity Brokering

* Validation of users created through user federation
* Validation of users created through identity brokering

