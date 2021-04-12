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

It would be possible to define more user profiles per realm, with client and scopes based rules to select appropriate user profile and required attributes for request. 
This mechanism allows flexibility like defining separate user profile for Admin console, which allows an admin to create an initial user with only specifying email, 
then requiring user to fill in first and last names on first login (where other user profile with more required attributes will be used).


## User Profile SPI

To allow user profiles to be customized an SPI will be introduced. 

The User Profile provider will be responsible for providing metadata around user profiles as well as validation of user profiles and update of the user mode. The API will not be covered in this design, but rather only the requirements for the API.

The User Profile provider needs to be supplied with the following context when validating:

* Is it a new user or update to existing user?
* What is the source of the information (user, admin, user federation, identity brokering)
* What scopes if any are being requested?
* Which client requested the action?

If validation is not passed the following information needs to be available from the provider:

* List of attributes that are not valid, with an error message per attribute
* List of missing attributes
* List of unsupported attributes

It should also be possible to only validate a set of attributes. An example use-case for this is an action that is asking for a single attribute, but is not able to request additional attributes if missing.

The user profile metadata provided by the provider should be able to answer the following questions within the context of the request:

* Is a specific attribute required?
* Is a specific attribute supported?
* Validations of the attribute value
* List of supported attributes


## Legacy Provider

For backwards compatibility a legacy provider will be implemented that hardcodes the current behavior of Keycloak. Existing realms will be configured to use the legacy provider.

The legacy provider will:

* Support username, firstname, lastname and email as mandatory (reflecting realm configurations like 'Email as username'), format validation only for email. Business validations on username and email uniqueness as now.
* Ignore unknown attributes in validations but copy them into user model - current behavior is that additional attributes needs to be validated elsewhere (i.e. through custom actions)
* Protect read-only attributes not to be updated/deleted on update


## Default Provider

The default provider will support defining an user profile configurations as a JSON document. Initially the admin console will simply allow editing the JSON directly, but the plan is to introduce an editor.

For new realms this provider will be used as the default provider. There will be a hardcoded user profiles config JSON (see format description later). A realm will use the hardcoded user profile configuration unless it is changed for the realm.

The user profiles configuration will be saved as JSON in a realm attribute. When editing the user profiles config for a realm the default hardcoded config will be shown, but it will only be saved in the realm attribute if the configuration is indeed modified and not equal to the built-in one. 

### Functionality design

#### What is User Profile
User Profile configuration defines a set of attributes in user profile and rules for them (mainly validations, visibility in the registration/update form, copying of the values into UserModel during update) to be applied when a given profile configuration is selected/requested for the processing.
* example of user profile configurations in one realm: basic user (basic info like email and name, basic T&C), paying customer (have to enter info necessary for billing, maybe additional T&C for purchased service etc), employee (have to enter additional info like employee number).
* user profile configurations are defined in realm (each realm has its own definition of profile configurations), there may be more profile configurations defined in realm. Default Keycloak distribution will define only one profile configuration used for the whole realm.
* as part of the configuration, attribute types are defined which allow configuring common validations to be shared in distinct user profiles configs (mainly attribute format validations tend to be the same in all user profile configs). These types are then used when attributes belonging to the user profile config are defined, and only few additional properties can be set to define attribute behavior in the profile config, mainly rules when the attribute is required. For more detail see JSON configuration design later. 
* It is allowed to configure additional/format validations for the special attributes `username` and `email`, but important business validations (uniqueness, requirement) are added automatically by the UserProfileProvider based on the realm configuration and context. It makes configuration simpler, easier, and less error prone.

#### How is User Profile Configuration selected
There must be mechanism how is user profile configuration selected/requested for distinct use cases/action, so correct validations are executed. This mechanism depend a lot on the concrete use case, so we should define them explicitly here and not bring them to the config.
* authentication flow 
    * default profile config for realm exists (it is named in the JSON profile config file itself)
    * default profile config can be named for client to override realm’s default. “Client Scopes” will be used for this assignment. Client scopes are different for OIDC and SAML clients, so two scopes have to be created if some profile config has to be used for both protocols, dot separated suffix like “.saml” can be added to the scope name for second one, eg. “profile.simple” and “profile.simple.saml”. “Client scopes” has to be created manually for user profile configs defined in the JSON config file. “Optional Client Scopes” can be used to name other user profile configs which may be requested by client over auth request param (see later).
    * then client can override it in auth request with protocol specific mechanism:
        * OIDC uses common scope parameter in OIDC auth request
            * `scope` to select user profile config is prefixed with `profile.` prefix, then name of the profile config as configured in JSON config. Eg. scope `profile.simple` selects user profile config named “simple” in the JSON config.
            * client should never ask for more user profile configs this way, if yes then only one is selected and used, but it is not guaranteed which one (“GUI order” from client scopes will be reflected in this selection, profile with lower number first)
            * OIDC clients can still use `scope` to ask for individual attributes as defined in the specification. Profiles JSON config allows you to configure that attribute is treated as required in this case and the user is forced to enter it. This mechanism is expected to be used in case when only one user profile configuration is used per realm to allow fine grained control by clients.
        * SAML - SAML auth request override is not required for now, may be implemented later by custom request param or by implementing ["SAML Protocol Extension for Requesting Attributes per Request"](https://www.oasis-open.org/committees/download.php/55790/Connectis%20_protocol_extension_draft.pdf) extension spec
        * only profile configs allowed for the client over “Optional Client Scopes” can be selected this way (see client configuration in previous point). If requested profile config doesn’t exist or is not allowed then client default is used (or realm default if client doesn’t have its default)
* Admin REST API user view/creation/update - may depend on who is calling the REST API (admin user, external app):
    * we are going to implement a mechanism which uses user profile configs for user create/update operations only (to validate data and populate/update UserModel), and selection of the used profile config is based on the client calling REST API (see next point). More detailed mechanism selecting profile configs based on the calling user’s roles/permissions and applied also to get/view operations (so distinct callers can see only some user attributes) is expected in the future.
    * User profile config is selected this way:
        * if the calling user is “Service Account” for some client from the realm, then user profile config is selected in the same way as if that client initiates auth flow - client default profile config is used if defined, or realm's default is used.
        * if the calling user is NOT “Service Account”, then the default profile config assigned to “security-admin-console” client is used (or realm's default if no one is assigned to this client).
* Admin Console 
    * Admin Console uses Admin REST API, so the mechanism defined for it and described in the previous point is used. It means that the user profile config assigned to the “security-admin-console” client is used or realm’s default if not configured.
* User Account console - one profile config is used for this app. Keycloak deployer has to decide which one will be used to show all important info and allow editing it. The app takes the default user profile config named for its client or realm’s default if not named for the client. 
    * New Account console is OnePage app and uses own REST API implemented in org.keycloak.services.resources.account.AccountRestService
* we need a new REST API endpoint which returns user profile config metadata so client apps can render forms, run client side validations etc (for use in Admin Console and new Account console and 3rd party apps). It should be a new API out of the Admin REST API as New User Account console doesn’t use Admin REST API.

#### How is User Profile Config used in the UserProfileProvider
* validate new user registration data from UI form/REST request before user is created
* control process of the UserModel population with attributes for new user creation (fill only attributes enabled for current profile config, ignore others)
* validate data from user update UI form/REST request before UserModel is updated
* control process of the UserModel update for existing user based on the new data (update only attributes enabled for current profile config, resolves problem which attributes have to be removed because they are editable for given profile config and no value is provided in new data - currently there is hardcoded list in Keycloak to control this behavior)
* validate if existing user matches some user profile config (to decide if user update action is necessary during auth flow, or to fill claims in OIDC response etc) - validators for this case can be a bit different than for data coming from form/REST request!

#### Other implementation notes
* We have to make really clear how username is available in validated data structure (cases like “email as username”) when validators are called for FRONTEND actions, so correct validators can be implemented
* For update case called in all contexts (UI form, REST API, Account app) existing UserModel (without applied changes) have to be available in validators so they can perform advanced business checks by comparing changes, having id of the user to check DB or external service etc.
* Actual user profile config, its annotations and configurations of all attributes in it are passed to the Freemarker context so crafted forms can use it (eg to implement dynamic forms, client side validations etc) now. Later it can be used to implement fully dynamic forms.
* Event log is extended and the name of the user profile config used for the action is stored there. Important for support.

### User Profile Config JSON configuration format

Design principles:
* validations can be defined only in attribute types to make things simpler
* only attribute types can inherit from other types to make things simpler, but still allow having common validations defined only once
* user profile configs then contain a list of attributes belonging to given profile config with assigned types, and allow only to control “required” validation for given attribute. If the attribute is not required then it is editable in the given profile config still (so validations are applied to it and is processed during UserModel update). Required validation configuration is simple (but provides field bindings to OAuth scopes still), it is expected that more profile configs will be defined if different required validations have to be used for distinct client apps.
* JSON free format `annotation` section is available in config of type, profile config and attribute to allow easier extensibility/customizations in UI/themes etc.

Field descriptions:
* `defaultProfileConfig` - name of the default user profile configuration (defined in `profileConfigs` section) for the realm 
* `types` - defines types of user profile attributes to be used/reused later in the `profileConfigs` section. Type name is key in this object (prevents duplicate types to be defined), can contain letters and numbers only. Type object can contain these configurations:
    * `parent` - allows to define parent type this one inherits from (see rules for other configs in case of inheritance, in general they are additive/updateable only)
    * `validations` - array of objects representing validations applied to the attribute of this type. In general these are mainly format/business validations etc. “required” validation is handled by setting in a profile config (see later). username and email fields can have some validations dependent on the realm setting (“email as username”, “email unique” etc) and they are not configured here to keep config simple (they will be added automatically by UserProfileProvider java code when necessary, needs to be documented in the doc!). Validation object contains:
    * `validator` - name/id of the Validator defined/implemented in Validation SPI (mandatory). If type inherits from parent type this is a key - validators from the parent type with the same name/id are not copied to the current type but new one is used instead. It is not possible to remove validation in sub-type.
    * `configuration` - configuration of the Validator for this validation (optional). Key value pairs, values can be text, number, boolean, array of values etc (will depend on Validator SPI). Each Validator defines its own possible configuration options.
    * `annotations` - optional JSON object containing unstructured key value pairs which are populated to the frontend frameworks (freemarker context, JavaScript, returned over user profile metadata REST API) and can be used as controls for UI/theme rendering etc. If type inherits from parent then keys in this object are primary id’s, they override respective config from higher level, it is not possible to remove key in sub-type.
* `profileConfigs` - definitions of user profile configurations which are then selected to control user validations and UserModel update. Profile config name is key in this object (prevents duplicate profile config names to be used), can contain letters and numbers only. Object bound to profile config name contains these configurations:
    * `attributes` - defines attributes present in the profile config. attribute name is key in this object (prevents duplicate attributes to be defined), can contain letters and numbers only. Attribute object can contain these configurations:
        * `type` - type of attribute from `types` section. This config is optional, type of the same name as the attribute name is used if not defined (existence of type have to be validated when JSON config is parsed). Configuration is validated to make sure that same named attribute from different profile configurations use the same type or subtypes only to prevent configuration error.
        * `required` - optional boolean default `false`. If `true` then “notBlank” validator is applied to this field - applied automatically by UserProfileProvider java code, default impl has to be extensible so custom subclasses can use another validator per field if necessary. `username` and `email` fields required validation may depend on the realm setting (“email as username”, “email unique” etc) and it is handled automatically by UserProfileProvider java code in this case, it overrides what is set here, needs to be well documented in the doc!
        * `requiredForAuthScopes` - array of strings, same as previous but field is required only if at least one of named OAuth scopes is requested for the current authentication flow - allows to use Keycloak with one user profile config only but working well with OAuth/OIDC scopes
        * `annotations` - optional JSON object containing unstructured key value pairs which are populated to the frontend frameworks (freemarker context, JavaScript, returned over user profile metadata REST API) and can be used as controls for UI/theme rendering etc. for the field/attribute. They are inherited from type, you can override or add new keys here (it is not possible to remove keys here)
    * `annotations` - optional JSON object containing unstructured key value pairs which are populated to the frontend frameworks (freemarker context, JavaScript, returned over user profile metadata REST API) and can be used as controls for UI/theme rendering etc. based on selected profile.

#### Example of the default configuration delivered with keycloak

Created to provide same functionality as current impl - improved a bit with some length validations to prevent DB constraint violations.

BTW, `password` field is not considered as part of the profile. In the current implementation of the Register flow and it is handled by a separate Authenticator, even its validation (password format policy, and equality of the “confirm password” field also). But it means that password validation is not performed together with other field validations, so password error messages can be shown after the next form submit after all other fields are patched - not the best UX. 

````
{
    "defaultProfileConfig": "default",
    "types": {
        "username": {
            "validations": [
                {
                    "validator": "length",
                    "configuration": {
                        "min": 3,
                        "max": 255
                    }
                }
            ]
        },
        "email": {
            "validations": [
                {
                    "validator": "length",
                    "configuration": {
                        "max": 255
                    }
                },
                {
                    "validator": "emailFormat"
                }
            ]
        },
        "firstName": {
            "validations": [
                {
                    "validator": "length",
                    "configuration": {
                        "max": 255
                    }
                }
            ]
        },
        "lastName": {
            "validations": [
                {
                    "validator": "length",
                    "configuration": {
                        "max": 255
                    }
                }
            ]
        }
    },
    "profileConfigs": {
        "default": {
            "attributes": {
                "username": {
                },
                "email": {
                    "required": true
                },
                "firstName": {
                    "required": true
                },
                "lastName": {
                    "required": true
                }
            }
        }
    }
}
````

#### Example of the more complex config file with more profile configs and annotations etc.

````
{
    "defaultProfileConfig": "simple",
    "types": {
        "username": {
            "validations": [
                {
                    "validator": "length",
                    "configuration": {
                        "min": 3,
                        "max": 80
                    }
                }
            ]
        },
        "email": {
            "validations": [
                {
                    "validator": "length",
                    "configuration": {
                        "max": 255
                    }
                },
                {
                    "validator": "emailFormat"
                },
                {
                    "validator": "emailDomainDenyList"
                }
            ],
            "annotations": {
                "formHintKey" : "userEmailFormFieldHint"    
            }
        },
        "emailForEmplyee": {
            "parent": "email",
            "validations": [
                {
                    "validator": "emailFromDomain",
                    "configuration": {
                        "domain": "acme.org"
                    }
                }
            ],
            "annotations": {
                "formHintKey" : "employeeEmailFormFieldHint"    
            }
        },
        "firstName": {
            "validations": [
                {
                    "validator": "length",
                    "configuration": {
                        "max": 255
                    }
                }
            ]
        },
        "lastName": {
            "validations": [
                {
                    "validator": "length",
                    "configuration": {
                        "max": 255
                    }
                }
            ]
        },
        "phone": {
            "validations": [
                {
                    "validator": "phoneNumberFormatInternational"
                }
            ]
        }
    },
    "profileConfigs": {
        "simple": {
            "attributes": {
                "username": {
                },
                "email": {
                    "required": true
                },
                "firstName": {
                    "annotations": {
                        "formAppearance":"whenValueExists"
                    }
                },
                "lastName": {
                    "annotations": {
                        "formAppearance":"whenValueExists"
                    }
                }
            },
            "annotations": {
                "profileFormHintKey" : "simpleProfileFormFieldHint"    
            }
        },
        "basic": {
            "attributes": {
                "username": {
                },
                "email": {
                    "required": true
                },
                "firstName": {
                    "required": true
                },
                "lastName": {
                    "required": true
                },
                "phone": {
                    "requiredForAuthScopes": ["phone", "phone2"]
                     }
            }
        },
        "employee": {
            "attributes": {
                "username": {
                },
                "email": {
                    "type": "emailForEmplyee",
                    "required": true,
                    "annotations": {
                        "formHintKey" : "employeeEmailFormFieldHint"    
                    }
                },
                "firstName": {
                    "required": true
                },
                "lastName": {
                    "required": true
                },
                "phone": {
                    "required": true
                }
            },
            "annotations": {
                "profileFormHintKey" : "emplyeeProfileFormFieldHint"    
            }
        }
    }
}
````

### Validations

This will allow using built-in validators as well as adding custom validators over Validation SPI. A validator can take an option config JSON object.

Built-in validators should cover:

* Name
* Email
* Pattern (regex)
* Number, with min and max
* Length, with min and max
* URL
* Date

### Annotations

Annotations are e.g. unstructured labels which are populated to the UI/freemarker context and can be considered for theme rendering.

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

This will enable use-cases such as not asking users for their birthdate until a client requests the user profile which requires birthdate. It will also enable adding additional required attributes, which users will be required to add on the next login.

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

There should be also mapper which allows to configure user profiles client app is interested in, and indicate to the client which of them current user already matches. 
Client app can use them to allow actions which require matched user profile levels, but forward user to Keycloak to ask user for additional attributes if action requires 
user profile which is not matched yet.


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

