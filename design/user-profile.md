# User Profile

* **Status**: Draft #1
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

If it is decided to support dynamic forms (see Dynamic Forms section below) the following fields would also be required:

* Label to use when displayed in forms
* UI order
* Input type

Ability to specify when an attribute is required should be quite flexible to cover use-cases like:

* Don't ask user for the users birthdate until a client requests the scope "birthdate"
* Allow an admin to create an initial user with only specifying email, then requiring user to fill in first and last names on first login
* Ask user for missing information after login with a social provider

A very rough idea on how the user profile JSON could look like:

    {
        "attributes": [{
            "name": "department",
            "permissions": {
                    "view": ["admin", "user"], 
                    "edit": ["admin"],
            }
            "requirement": {
                "always",
            }
            "validation": {
                "name", 
                "context" : ["UserProfileUpdate"],
                { "length" : { "min" : 10, "max": 20 } },
            },
            "converter": {
                "datetime-converter": {
                    "datetime-format" : "tt/mm/yyyy"
                },
                "timezone-converter": {
                    "z": "UTC-0"
                };
            },
            "annotations":  {
                "key": "value",  
                "framecolor": "red",
                "gui-order": "1",
                "type": "dropdown",
                "defaults" : [1,2,3,5]
            }
        }]
    }

### Required Fields

Requirement should support following values:

* `optional` (default if no value given)
* `always`
* `user`
* `{ "scope" : "scope-1" }`
* `{ "scope" : [ "scope-1", "scope-2" ] }`

The `user` requirement allows an admin to create a user without specifying the value, while the user will then be required to enter the value on first login.

The `scope` requirement allows an attribute to only be required when a specific client scope is being requested.

### Validation

Validation should support the following values:

* `validator-id`
* `{ "validator-id" : "<config>" }`
* `{ "validator-id" : { ... } }`

This will allow using built-in validators as well as adding custom validators. A validator can take an option config either as a single field, or a JSON object.

Built-in validators should cover:

* Name
* Email
* Pattern (regex)
* Number, with min and max
* Length, with min and max
* URL
* Date

There will be a user attribute SPI allowing custom validators to be created.

Validators can also be attached to a special context meaning the validation will only be performed in that specific context.
This allows distinguishing admin and user validation. For example, an admin may create a user without first and last name, but the user must provider these upon login.

Possible contexts include:

* UpdateProfile
* UserRegistration
* UserResource
* Account
* Registration

### Converter (optional)

Converters can be used to preprocess an attribute value before storing it.
A converter may also affect the "read" process.

### Annotations

Annotations are e.g. unstructured labels which are populated to the freemarker context and can be considered for theme rendering.

## Built-in Attributes

Keycloak should add more attributes out of the box than it does today. 

This means defining the standard key for the attribute as well as optionally rendering in forms. Having standard names for attributes will further help on standardising and simplification of configuring Keycloak. Of course the attributes defined should be flexible and even though "birthdate" is used as a standard attribute name, that can be changed to dob if wanted by simply modifying the default profile.

Standard attribute names will be based on [claims from OpenID Connect Core](https://openid.net/specs/openid-connect-core-1_0.html#StandardClaims), but we may not cover all attributes listed there at least initially. Especially attributes such as picture are problematic as the ideal experience is one where users are able to upload a picture through admin and account consoles rather than simply listing an externally managed URL.


## Dynamic Forms

With regards to forms there is two options we can choose from:

1. Crafted forms - craft forms such as registration form, where fields are shown or hidden based on user profile. To add custom attributes the form would be extended.
2. Dynamic forms - add additional metadata to user profile attributes so forms can be dynamically generated.

The crafted forms may be the ideal solution as that would allow us to more carefully craft form with good usability. A dynamic form may only have limited use as most likely when custom
attributes are required it would be best to create a custom form to be able to have greater control of the layout.


## Wiring

This section will discuss how the user profile will be used within Keycloak. 

### Registration

The registration form today has a very limited number of attributes built-in. Custom validation is also complicated as it requires introducing a custom registration flow.

We will expand the registration form either with a crafted form with additional attributes or a dynamic form generated from the user profile.

Validation will also be driven by the user profile making it much easier to introduce custom attributes and associated validation.

If validation doesn't pass a generic error message will be displayed at the top, with individual error messages associated with each input field.

### Actions

The update profile action will be expanded to cover attributes in a similar fasion to the registration form. Further, it will support only asking for the fields that are not currently passing validation. 

This will enable use-cases such as not asking users for their birthdate until a client requests the scope birthdate. It will also enable adding additional required attributes, which users will be required to add on the next login.

### Account Console

The new Account Console will display fields as defined the user profile, as well as use the profile to validate and display individual messages.

The old Account Console will require custom forms to add additional fields, it will use the profile to validate, but will not display individual messages.

### Admin Console

The user profile will display fields as defined in the user profile. Adding additional attributes will only allow selecting attributes from the profile and not add arbritary attributes.

### User Federation

This section is incomplete and is mainly a brief suggestion of what we could do.

User federation mappers that map to user attributes will only allow selecting attributes from the user profile, not allowing mapping to arbritary attributes, unless the user profile explicitly allows arbritary attributes (see legacy provider).

Depending on the user federation provider configuration we may want to ask users to enter missing attributes using the update profile action, rejecting the user, or simply ignoring that the user does not validate. The latter may not be ideal, but may be required.

We could also have a verification on whether or not all mandatory attributes have been mapped for a user federation provider.

### Identity Brokering

This section is incomplete and is mainly a brief suggestion of what we could do.

User federation mappers that map to user attributes will only allow selecting attributes from the user profile, not allowing mapping to arbritary attributes, unless the user profile explicitly allows arbritary attributes (see legacy provider).

We can ask users to enter missing attributes using the update profile action. Today we don't have support for transient users (or users fully managed in the external IdP). If we add that then we will not want to ask users for missing attributes as those attributes would be lost on the next login.

We could also have a verification on whether or not all mandatory attributes have been mapped for a identity provider.

### Protocol Mappers

All mappers that use user attributes should provide a drop-down with list of attributes and no longer have a input field to specify the attribute.


## Milestones

### M1 SPI, Legacy Provider and UserModel preparation/refactoring

Define the SPI/API for the user profile, its attributes with validators (and optionally permissions).
Implement simple legacy provider.

Migrate to new user model where all fields like username, email, etc. are attributes
Don't change database model: Use entity - model adapter classes to enforce legacy behaviour: all legacy fields are "mandatory"
Use legacy provider to enforce old behaviour

(Potentially, those can be split into several contributions)

### M2 Validation

Focus for the first milestone should be validation of users self-registering and updates to the account through account console and actions.

This will require the legacy provider, and until the default provider is added users will need to add a custom user profile provider to validate additional attributes.

### M3 Default Provider with additional attributes

Add default provider with basic text-area to edit the profile in the admin console. 
Registration form, account console and admin console will be expanded depending on what is decided in the dynamic forms section.
Introduce client-scope based control of requirement configuration.

### M4 Mappers

Update user attribute mappers to select attributes from user profile. 

### M5 User Profile editor

Editor in admin console for defining user profile.

### M6 User Federation & Identity Brokering

Integrate user federation and identity brokering with user profile. Scope to be decided as the correspodning sections above are completed.

